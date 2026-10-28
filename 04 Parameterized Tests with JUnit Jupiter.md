# Parameterized Tests with JUnit Jupiter

## Learning objectives

By the end of this topic, you will be able to:

- explain why and when a test should be parameterized;
- write a parameterized test with `@ParameterizedTest`;
- explain how an argument source creates separate test invocations;
- use `@ValueSource`, `@CsvSource`, `@MethodSource`, and `@EnumSource`;
- use `@NullSource`, `@EmptySource`, and `@NullAndEmptySource` for common edge cases;
- choose an argument source that matches the shape of the test data;
- explain basic implicit argument conversion;
- create readable invocation display names;
- recognize when separate test methods are clearer than one parameterized test.

---

## 1. Why parameterize a test?

Sometimes several test cases exercise exactly the same behavior and differ only in their input or expected result.

Consider a utility that classifies an integer as positive:

```java
public final class NumberClassifier {

    private NumberClassifier() {
    }

    public static boolean isPositive(int number) {
        return number > 0;
    }
}
```

We could test several positive values with separate methods:

```java
@Test
void shouldRecognizeOneAsPositive() {
    assertTrue(NumberClassifier.isPositive(1));
}

@Test
void shouldRecognizeTenAsPositive() {
    assertTrue(NumberClassifier.isPositive(10));
}

@Test
void shouldRecognizeMaximumIntegerAsPositive() {
    assertTrue(NumberClassifier.isPositive(Integer.MAX_VALUE));
}
```

The scenarios use different inputs, but their test logic is identical. Copying the method creates noise and makes adding another representative value unnecessarily expensive.

A parameterized test separates the **test logic** from the **test data**:

```java
import static org.junit.jupiter.api.Assertions.assertTrue;

import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;

class NumberClassifierTest {

    @ParameterizedTest
    @ValueSource(ints = {1, 10, Integer.MAX_VALUE})
    void shouldRecognizePositiveNumber(int number) {
        assertTrue(NumberClassifier.isPositive(number));
    }
}
```

JUnit runs `shouldRecognizePositiveNumber()` three times:

```text
shouldRecognizePositiveNumber(int)
├─ [1] number = 1
├─ [2] number = 10
└─ [3] number = 2147483647
```

Each run is a separate **invocation**. If the value `10` produces a failure, the other invocations still have their own results.

Parameterized tests are useful because they can:

- remove mechanical duplication;
- make groups of equivalent examples easy to scan;
- encourage representative boundary and invalid-input cases;
- make it easy to add another data row without copying test logic;
- report exactly which argument set failed.

However, reducing the number of lines is not the main goal. The parameterized form is better only when it communicates the scenarios more clearly.

---

## 2. One scenario, many data sets

A good parameterized test expresses **one behavioral rule** for several data sets.

For example:

> A username consisting only of whitespace is rejected.

The strings `" "`, `"   "`, `"\t"`, and `"\n"` are different examples of the same category. They can share one test body.

By contrast, these are different behaviors:

- a blank username is rejected;
- a valid username is normalized to lowercase;
- a duplicate username causes `DuplicateUserException`;
- a successfully registered user is stored.

All four concern registration, but their arrangements, actions, and expectations differ. Combining them into one parameterized test would hide those differences.

A useful decision rule is:

> Parameterize when the Arrange–Act–Assert structure remains essentially the same and only the data changes.

If each row requires a different branch in the test body, the rows probably do not describe one scenario.

---

## 3. Anatomy of a parameterized test

A basic parameterized test contains three parts:

1. `@ParameterizedTest` marks a method that JUnit should invoke repeatedly.
2. An argument source supplies one or more argument sets.
3. Method parameters receive the arguments for the current invocation.

```java
@ParameterizedTest
@ValueSource(strings = {"java", "junit", "testing"})
void shouldRecognizeLowercaseWord(String word) {
    assertEquals(word, word.toLowerCase());
}
```

The source contains three values. The method has one parameter. JUnit maps one source value to that parameter for each invocation.

### 3.1 `@ParameterizedTest`, not `@Test`

`@ParameterizedTest` comes from a different package than ordinary Jupiter annotations:

```java
import org.junit.jupiter.params.ParameterizedTest;
```

Do not replace it with `@Test`, and do not put both annotations on the same method:

```java
// Incorrect design
@Test
@ParameterizedTest
@ValueSource(ints = {1, 2, 3})
void testNumber(int number) {
    // ...
}
```

An ordinary `@Test` method does not receive values from an argument source. A parameterized method is a different kind of Jupiter test method.

### 3.2 Every parameterized test needs a source

This method is incomplete:

```java
@ParameterizedTest
void shouldAcceptCode(String code) {
    // ...
}
```

JUnit has no arguments with which to invoke it. Add at least one source annotation:

```java
@ParameterizedTest
@ValueSource(strings = {"A1", "B2"})
void shouldAcceptCode(String code) {
    // ...
}
```

### 3.3 Arguments and parameters must correspond

For ordinary direct argument consumption, source values map to method parameters by position:

```java
@ParameterizedTest
@CsvSource({
        "2, 3, 5",
        "10, 7, 17"
})
void shouldAddNumbers(int left, int right, int expected) {
    assertEquals(expected, left + right);
}
```

The first column goes to `left`, the second to `right`, and the third to `expected`.

The number and types of supplied arguments must be compatible with the parameters. Otherwise, the invocation fails before the test body can run.

### 3.4 Required dependency

Parameterized-test APIs belong to the `junit-jupiter-params` artifact. If the project uses the dependency-aggregating `org.junit.jupiter:junit-jupiter` artifact from the previous topic, parameterized-test support is already included.

Projects that manage Jupiter modules individually need `junit-jupiter-params` explicitly. The version must match the other JUnit modules:

```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter-params</artifactId>
    <version>${junit.version}</version>
    <scope>test</scope>
</dependency>
```

Do not add this second declaration when the project's existing JUnit dependency already supplies it.

---

## 4. Separate invocations and lifecycle behavior

A parameterized method is written once, but JUnit reports each argument set separately. One failing invocation does not turn the complete method into an indistinguishable single case.

Each invocation also follows the normal test lifecycle:

```text
@BeforeEach
invocation 1
@AfterEach

@BeforeEach
invocation 2
@AfterEach

@BeforeEach
invocation 3
@AfterEach
```

Consider this class:

```java
class ShoppingCartTest {

    private ShoppingCart cart;

    @BeforeEach
    void setUp() {
        cart = new ShoppingCart();
    }

    @ParameterizedTest
    @ValueSource(ints = {1, 2, 10})
    void shouldAddRequestedQuantity(int quantity) {
        cart.add("BOOK", quantity);

        assertEquals(quantity, cart.itemCount());
    }
}
```

Each quantity is tested with a fresh cart. This preserves the isolation discussed in the previous topic.

Arguments supplied to a parameterized test method are not automatically available as parameters of `@BeforeEach` or `@AfterEach` methods. Keep argument-specific arrangement in the parameterized method unless an advanced extension is deliberately used.

---

## 5. `@ValueSource`: one simple argument

`@ValueSource` is the simplest source for literal values. It supplies exactly one argument per invocation.

```java
@ParameterizedTest
@ValueSource(strings = {"admin", "support", "student"})
void shouldAcceptKnownRole(String role) {
    assertTrue(RoleValidator.isKnown(role));
}
```

The annotation supports arrays of these literal types:

- `short`, `byte`, `int`, and `long`;
- `float` and `double`;
- `char` and `boolean`;
- `String`;
- `Class`.

The annotation attribute matches the literal type:

```java
@ValueSource(ints = {0, 1, 100})
@ValueSource(longs = {1L, 1_000_000L})
@ValueSource(doubles = {0.0, 0.5, 1.0})
@ValueSource(chars = {'a', 'Z'})
@ValueSource(booleans = {true, false})
@ValueSource(strings = {"java", "junit"})
@ValueSource(classes = {String.class, Integer.class})
```

Only one of these attributes is used on a particular `@ValueSource` annotation.

### 5.1 When `@ValueSource` fits

Use it when:

- each invocation needs one value;
- values can be written as annotation literals;
- every invocation has the same expectation or can derive it without hiding the scenario.

This is a good use:

```java
@ParameterizedTest
@ValueSource(ints = {-1, -10, Integer.MIN_VALUE})
void shouldRecognizeNegativeNumber(int number) {
    assertTrue(number < 0);
}
```

This is not a good fit for `@ValueSource`:

```text
input: 2, expected: 4
input: 3, expected: 9
input: 10, expected: 100
```

Each invocation needs two related values. `@CsvSource` or `@MethodSource` represents that shape more directly.

### 5.2 What `@ValueSource` does not provide

`@ValueSource` does not directly provide:

- `null`;
- several arguments per invocation;
- arbitrary domain objects;
- values created by running Java code.

Other sources cover those cases.

---

## 6. Null, empty, and blank input

Null and empty values frequently belong to validation boundaries. Jupiter provides compact sources for parameterized tests with one argument.

### 6.1 `@NullSource`

`@NullSource` supplies one `null` argument:

```java
@ParameterizedTest
@NullSource
void shouldRejectNullUsername(String username) {
    assertFalse(UsernameValidator.isValid(username));
}
```

It cannot supply `null` to a primitive parameter such as `int`, because Java primitives cannot hold `null`.

### 6.2 `@EmptySource`

`@EmptySource` supplies an appropriate empty value. For a `String`, that value is `""`:

```java
@ParameterizedTest
@EmptySource
void shouldRejectEmptyUsername(String username) {
    assertFalse(UsernameValidator.isValid(username));
}
```

It also supports several collection, map, and array types. At Fundamentals level, its most common use is empty-string validation.

### 6.3 `@NullAndEmptySource`

`@NullAndEmptySource` combines the preceding sources:

```java
@ParameterizedTest
@NullAndEmptySource
void shouldRejectMissingUsername(String username) {
    assertFalse(UsernameValidator.isValid(username));
}
```

The method is invoked once with `null` and once with `""`.

### 6.4 Empty is not the same as blank

An empty string has length zero. A blank string is empty or consists only of whitespace. `@EmptySource` supplies the empty string but does not supply all possible blank strings.

Sources can be combined:

```java
@ParameterizedTest
@NullAndEmptySource
@ValueSource(strings = {" ", "   ", "\t", "\n"})
void shouldRejectNullEmptyAndBlankUsername(String username) {
    assertFalse(UsernameValidator.isValid(username));
}
```

This is one coherent validation rule applied to several representatives of invalid input.

---

## 7. `@CsvSource`: several columns per invocation

`@CsvSource` is useful when each invocation needs multiple related values. Each record represents one argument set, and columns map to method parameters by position.

Suppose the production code calculates a percentage:

```java
public final class PercentageCalculator {

    private PercentageCalculator() {
    }

    public static int percentageOf(int value, int percentage) {
        return value * percentage / 100;
    }
}
```

A compact table of examples can be written directly in the annotation:

```java
import static org.junit.jupiter.api.Assertions.assertEquals;

import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;

class PercentageCalculatorTest {

    @ParameterizedTest
    @CsvSource({
            "100, 10, 10",
            "200, 25, 50",
            "80,  50, 40",
            "0,   90, 0"
    })
    void shouldCalculatePercentage(
            int value,
            int percentage,
            int expected
    ) {
        int actual = PercentageCalculator.percentageOf(value, percentage);

        assertEquals(expected, actual);
    }
}
```

The source visually resembles a small data table:

| `value` | `percentage` | `expected` |
| ---: | ---: | ---: |
| 100 | 10 | 10 |
| 200 | 25 | 50 |
| 80 | 50 | 40 |
| 0 | 90 | 0 |

### 7.1 Basic conversion

CSV records are text, but the method above declares `int` parameters. Jupiter converts compatible textual values into the requested types before calling the test method.

This makes it possible to use parameters such as:

- primitive numbers and their wrapper types;
- `boolean` and `char`;
- enum types;
- common Java types supported by Jupiter's converters.

Conversion should make a table easier to read. If the conversion is surprising or requires complicated rules, use `@MethodSource` and construct correctly typed objects in Java.

### 7.2 Quoting values that contain a comma

By default, a comma separates columns. A string containing a comma must be quoted with single quotes inside the CSV record:

```java
@ParameterizedTest
@CsvSource({
        "'London, UK', Europe",
        "'Lima, Peru', South America"
})
void shouldReadLocation(String location, String region) {
    assertFalse(location.isBlank());
    assertFalse(region.isBlank());
}
```

Without the inner quotes, `London, UK, Europe` would be interpreted as three columns.

### 7.3 Empty text and `null` are different

With the default CSV syntax:

```java
@CsvSource({
        "apple, ''",
        "banana,"
})
```

- `''` represents an empty string;
- an unquoted empty column represents `null`.

This distinction matters when validation treats empty and missing values differently. A `null` CSV value cannot be passed to a primitive parameter.

### 7.4 Text blocks for larger inline tables

Java text blocks can make several rows easier to align:

```java
@ParameterizedTest(name = "{0} with quantity {1} costs {2}")
@CsvSource(useHeadersInDisplayName = true, textBlock = """
        PRODUCT, QUANTITY, EXPECTED_PRICE
        book,    1,        1500
        book,    2,        3000
        pen,     5,        1000
        """)
void shouldCalculateLinePrice(
        String product,
        int quantity,
        int expectedPrice
) {
    assertEquals(
            expectedPrice,
            PriceList.linePrice(product, quantity)
    );
}
```

Use an inline CSV table only while it remains compact and understandable. A very large table can bury the behavior under data and may belong in another source or another kind of test.

---

## 8. `@MethodSource`: data created by Java code

`@MethodSource` refers to a factory method that supplies arguments. It is the most flexible common source in this topic.

Use it when:

- an invocation needs domain objects;
- values cannot be expressed as annotation literals;
- input data is clearer when created with constructors or factory methods;
- arguments are calculated or assembled programmatically;
- each invocation needs several values and CSV text would be fragile.

### 8.1 One argument per invocation

A factory can return a stream of values:

```java
import static org.junit.jupiter.api.Assertions.assertTrue;

import java.util.stream.Stream;

import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.MethodSource;

class EmailValidatorTest {

    @ParameterizedTest
    @MethodSource("validEmails")
    void shouldAcceptValidEmail(String email) {
        assertTrue(EmailValidator.isValid(email));
    }

    static Stream<String> validEmails() {
        return Stream.of(
                "student@example.com",
                "first.last@example.org",
                "user+label@example.net"
        );
    }
}
```

The name in `@MethodSource("validEmails")` identifies the factory method.

If the source name is omitted, Jupiter looks for a factory with the same name as the parameterized test method:

```java
@ParameterizedTest
@MethodSource
void shouldAcceptValidEmail(String email) {
    assertTrue(EmailValidator.isValid(email));
}

static Stream<String> shouldAcceptValidEmail() {
    return Stream.of("student@example.com", "user@example.org");
}
```

The explicit name is often easier for beginners to recognize and maintain.

### 8.2 Several arguments with `Arguments`

For multiple parameters, return argument sets:

```java
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.params.provider.Arguments.arguments;

import java.time.LocalDate;
import java.util.stream.Stream;

import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.Arguments;
import org.junit.jupiter.params.provider.MethodSource;

class SubscriptionTest {

    @ParameterizedTest
    @MethodSource("expirationCases")
    void shouldDetermineWhetherSubscriptionIsExpired(
            Subscription subscription,
            LocalDate today,
            boolean expected
    ) {
        assertEquals(expected, subscription.isExpiredOn(today));
    }

    static Stream<Arguments> expirationCases() {
        return Stream.of(
                arguments(
                        new Subscription(LocalDate.of(2026, 7, 31)),
                        LocalDate.of(2026, 7, 30),
                        false
                ),
                arguments(
                        new Subscription(LocalDate.of(2026, 7, 31)),
                        LocalDate.of(2026, 8, 1),
                        true
                )
        );
    }
}
```

Each call to `arguments(...)` creates the complete argument set for one invocation.

### 8.3 The factory is normally `static`

A local `@MethodSource` factory must normally be `static` because Jupiter uses the default `PER_METHOD` test instance lifecycle:

```java
static Stream<Arguments> expirationCases() {
    // ...
}
```

With `@TestInstance(TestInstance.Lifecycle.PER_CLASS)`, a local source method may be non-static. As discussed in the lifecycle topic, do not select `PER_CLASS` merely to remove `static`; it also makes instance state shared between test methods.

An external factory method must be `static`.

### 8.4 Keep factories readable

A factory method is test code. It should make the cases easier to understand, not conceal them behind complicated calculations or external I/O.

Avoid sources that:

- query a production database;
- call a remote service;
- depend on the current time;
- produce random cases without a controlled seed;
- read a large file for an ordinary unit test;
- contain complex logic that can itself be wrong.

If the source generates expected results by copying the production algorithm, both can repeat the same defect and make the test misleading.

---

## 9. `@EnumSource`: enum constants as arguments

`@EnumSource` supplies constants from an enum type.

```java
enum PaymentStatus {
    CREATED,
    AUTHORIZED,
    CAPTURED,
    CANCELLED
}
```

To test every constant:

```java
@ParameterizedTest
@EnumSource(PaymentStatus.class)
void shouldHaveUserFacingLabel(PaymentStatus status) {
    assertFalse(statusLabel(status).isBlank());
}
```

JUnit invokes the method once for each `PaymentStatus` constant.

### 9.1 Select named constants

Sometimes only a subset belongs to one rule:

```java
@ParameterizedTest
@EnumSource(
        value = PaymentStatus.class,
        names = {"AUTHORIZED", "CAPTURED"}
)
void shouldRepresentSuccessfulProcessing(PaymentStatus status) {
    assertTrue(PaymentRules.isSuccessful(status));
}
```

### 9.2 Exclude named constants

You can select every constant except a named subset:

```java
@ParameterizedTest
@EnumSource(
        value = PaymentStatus.class,
        names = "CANCELLED",
        mode = EnumSource.Mode.EXCLUDE
)
void shouldAllowStatusToAppearInActiveHistory(PaymentStatus status) {
    assertTrue(PaymentRules.isVisibleInActiveHistory(status));
}
```

Use filtering only when the resulting group has a clear business meaning. A long and obscure filter can make separate explicit tests easier to understand.

### 9.3 Why `@EnumSource` is better than copying names

This is usually preferable:

```java
@EnumSource(PaymentStatus.class)
```

to this:

```java
@ValueSource(strings = {"CREATED", "AUTHORIZED", "CAPTURED", "CANCELLED"})
```

`@EnumSource` supplies typed enum values and automatically follows the enum when a new constant is added. Whether that new constant should pass the existing rule still requires a human decision, but the test is less likely to silently ignore it.

---

## 10. Basic argument conversion

An argument source and a test parameter do not always have exactly the same runtime type. Jupiter supports several built-in conversions.

### 10.1 Widening primitive conversion

An `int` value can be supplied to a compatible wider parameter:

```java
@ParameterizedTest
@ValueSource(ints = {1, 2, 3})
void shouldConvertToLong(long value) {
    assertTrue(value > 0L);
}
```

### 10.2 Conversion from strings

Text-based sources such as `@CsvSource` can be converted according to the declared parameter type:

```java
import static org.junit.jupiter.api.Assertions.assertEquals;

import java.time.DayOfWeek;
import java.time.LocalDate;

import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;

@ParameterizedTest
@CsvSource({
        "2026-07-20, MONDAY",
        "2026-07-21, TUESDAY"
})
void shouldDetermineDayOfWeek(
        LocalDate date,
        DayOfWeek expected
) {
    assertEquals(expected, date.getDayOfWeek());
}
```

JUnit converts the ISO date text to `LocalDate` and the enum name to `DayOfWeek`.

At this level, you need to understand the principle:

> Jupiter uses the declared parameter type to determine whether a supplied argument can be converted.

Custom converters and argument aggregation are useful advanced features, but they are outside the core scope of this topic. When basic conversion is insufficient, `@MethodSource` often provides the clearest Fundamentals-level solution.

### 10.3 Do not hide incorrect data behind conversion

If a value cannot be converted, the invocation fails during argument resolution:

```java
@ParameterizedTest
@CsvSource("not-a-number, 10")
void shouldAdd(int left, int right) {
    // the test body cannot receive "not-a-number" as int
}
```

This is a test configuration failure, not a failure of the production behavior.

---

## 11. Display names for invocations

Useful invocation names make failures easier to identify. By default, JUnit reports an index and representations of the current arguments.

A custom pattern can be supplied through the `name` attribute:

```java
@ParameterizedTest(name = "[{index}] {0} × {1} = {2}")
@CsvSource({
        "2, 3, 6",
        "5, 4, 20",
        "0, 9, 0"
})
void shouldMultiply(int left, int right, int expected) {
    assertEquals(expected, left * right);
}
```

Typical placeholders include:

| Placeholder | Meaning |
| --- | --- |
| `{index}` | Current invocation number, starting from 1 |
| `{arguments}` | All arguments |
| `{0}`, `{1}`, ... | One argument by its zero-based position |
| `{displayName}` | Display name of the parameterized method |

The example produces names similar to:

```text
[1] 2 × 3 = 6
[2] 5 × 4 = 20
[3] 0 × 9 = 0
```

### 11.1 Name the scenario, not the mechanism

This name adds little information:

```java
@ParameterizedTest(name = "test {index} with arguments {arguments}")
```

Prefer a name that exposes the relationship being tested:

```java
@ParameterizedTest(name = "quantity {0} produces total {1}")
```

Do not force every short parameterized test to have a custom pattern. Default names are often sufficient when arguments already explain the case.

### 11.2 JUnit 6 quoting of text arguments

JUnit 6 quotes text-based arguments in parameterized-test display names by default and escapes special characters. For example, an empty string is displayed as `""` and a newline as `"\n"`. This makes invisible and whitespace values easier to distinguish in reports.

The display name is diagnostic output. Assertions still determine whether the invocation passes or fails.

---

## 12. Choosing an argument source

Choose the smallest source that represents the data clearly.

| Need | Recommended source | Example |
| --- | --- | --- |
| One literal value per invocation | `@ValueSource` | several invalid integer values |
| One `null` argument | `@NullSource` | missing username |
| One empty string, collection, or array | `@EmptySource` | empty search query |
| Both `null` and empty | `@NullAndEmptySource` | absent text input |
| Several simple columns | `@CsvSource` | input, rate, expected result |
| Objects or programmatically built data | `@MethodSource` | orders, dates, domain records |
| Enum constants | `@EnumSource` | all order statuses |

A progression that often works is:

1. Start with `@ValueSource` for one simple value.
2. Move to `@CsvSource` when each invocation needs several simple columns.
3. Use `@MethodSource` when the table contains objects or becomes awkward as text.
4. Use `@EnumSource` when the argument set is naturally defined by an enum.

Do not choose `@MethodSource` merely because it is the most flexible. Extra flexibility also means more code and more places to hide a mistake.

---

## 13. When parameterized tests improve a suite

Parameterized tests are usually a good choice when all of the following are true:

- the same production behavior is exercised;
- the same action is performed;
- the same assertion structure is used;
- data sets can be understood as representatives of one category or rule;
- a failed invocation clearly identifies the broken case.

For example, several unsupported discount percentages can share one exception test:

```java
@ParameterizedTest
@ValueSource(ints = {-1, 101, 150})
void shouldRejectPercentageOutsideRange(int percentage) {
    assertThrows(
            IllegalArgumentException.class,
            () -> new Discount(percentage)
    );
}
```

The inputs all violate the same rule: valid percentage is from 0 through 100.

### 13.1 Keep separate tests for different boundaries when meaning differs

The values `-1` and `101` are both invalid, but the lower and upper boundaries may deserve separate methods if they lead to different exceptions, messages, or business explanations.

Clarity is more important than eliminating two short methods.

### 13.2 Avoid branching by expected scenario

This is a warning sign:

```java
@ParameterizedTest
@CsvSource({
        "valid@example.com, true",
        "missing-at-sign, false"
})
void testEmail(String email, boolean expectedValid) {
    if (expectedValid) {
        assertTrue(EmailValidator.isValid(email));
    } else {
        assertFalse(EmailValidator.isValid(email));
    }
}
```

The branch is unnecessary even here—`assertEquals(expectedValid, actual)` would be simpler—but mixing valid and invalid categories may still make failures less expressive.

Often two tests communicate intent better:

```java
@ParameterizedTest
@ValueSource(strings = {"student@example.com", "first.last@example.org"})
void shouldAcceptValidEmail(String email) {
    assertTrue(EmailValidator.isValid(email));
}

@ParameterizedTest
@ValueSource(strings = {"missing-at-sign", "@missing-local-part"})
void shouldRejectInvalidEmail(String email) {
    assertFalse(EmailValidator.isValid(email));
}
```

Each method now states one rule.

### 13.3 A large data table is not automatically a unit test

Parameterized tests can run many data rows, but a huge external data set does not automatically improve a unit test. Ask:

- Are the rows representative or merely numerous?
- Can a reader understand why each row exists?
- Are boundaries and equivalence classes visible?
- Would a smaller set provide the same evidence?
- Does loading the data introduce I/O or nondeterminism?

Good test selection matters more than row count.

---

## 14. Common mistakes

### 14.1 Using `@Test` instead of `@ParameterizedTest`

An ordinary test does not consume arguments from `@ValueSource` or another parameterized source.

### 14.2 Forgetting the source

`@ParameterizedTest` alone provides no invocation arguments.

### 14.3 Providing the wrong number of columns

If a CSV row has two columns but the method expects three source parameters, argument resolution fails.

### 14.4 Using `@ValueSource` for several parameters

`@ValueSource` provides one argument per invocation. Use `@CsvSource` or `@MethodSource` for several related arguments.

### 14.5 Passing `null` to a primitive

`null` cannot be assigned to `int`, `boolean`, or another primitive parameter. Use a reference type only when `null` is meaningful.

### 14.6 Forgetting CSV quoting

A comma inside an unquoted string creates another column. Quote such a field or use a different representation.

### 14.7 Making a local method source non-static under `PER_METHOD`

Under the default lifecycle, a local factory method must be `static`. Do not change the complete test instance lifecycle only to avoid that keyword.

### 14.8 Creating expectations with production logic

This is weak:

```java
static Stream<Arguments> cases() {
    return inputs().map(input -> arguments(input, ProductionCode.calculate(input)));
}
```

The expected result comes from the same code being tested. A defect can reproduce itself in the expectation.

### 14.9 Combining unrelated behaviors

A method with many boolean flags, conditional setup, several alternative actions, and different assertion branches is usually several tests disguised as one.

### 14.10 Adding too many parameters

A method with eight primitive and string parameters is hard to read and easy to map incorrectly. Consider a domain object, a small test record supplied by `@MethodSource`, or separate scenarios.

### 14.11 Treating all values as strings

Prefer typed parameters such as `LocalDate`, enum values, and numbers when they express the domain. Do not parse everything manually inside the test without a reason.

### 14.12 Hiding the meaning of rows

Data like `"1, 2, 3, false"` may be compact but unclear. Use an explanatory display name, headers, named objects, or separate tests when a reader cannot infer the scenario.

---

## 15. Worked example: refactoring repeated tests

Suppose a delivery service applies these fees:

- order totals below 5,000 cents cost 700 cents to deliver;
- order totals from 5,000 through 9,999 cents cost 300 cents;
- order totals of 10,000 cents or more have free delivery.

```java
public final class DeliveryFeeCalculator {

    public int calculate(int orderTotal) {
        if (orderTotal < 0) {
            throw new IllegalArgumentException("orderTotal must not be negative");
        }
        if (orderTotal < 5_000) {
            return 700;
        }
        if (orderTotal < 10_000) {
            return 300;
        }
        return 0;
    }
}
```

The original tests repeat the same action:

```java
@Test
void shouldChargeSevenHundredBelowFirstThreshold() {
    DeliveryFeeCalculator calculator = new DeliveryFeeCalculator();

    int actual = calculator.calculate(4_999);

    assertEquals(700, actual);
}

@Test
void shouldChargeThreeHundredAtFirstThreshold() {
    DeliveryFeeCalculator calculator = new DeliveryFeeCalculator();

    int actual = calculator.calculate(5_000);

    assertEquals(300, actual);
}

@Test
void shouldProvideFreeDeliveryAtSecondThreshold() {
    DeliveryFeeCalculator calculator = new DeliveryFeeCalculator();

    int actual = calculator.calculate(10_000);

    assertEquals(0, actual);
}
```

### Step 1: identify the invariant test logic

Every test:

1. creates the calculator;
2. calls `calculate(orderTotal)`;
3. compares the result with an expected fee.

Only `orderTotal` and `expectedFee` change.

### Step 2: choose a source

Each invocation needs two simple numeric columns. `@CsvSource` is a natural fit.

### Step 3: preserve boundary meaning

Do not choose arbitrary rows only to increase their number. Include values around the boundaries:

| Order total | Why it matters | Expected fee |
| ---: | --- | ---: |
| 0 | lowest valid total | 700 |
| 4,999 | just below first threshold | 700 |
| 5,000 | first threshold | 300 |
| 9,999 | just below free delivery | 300 |
| 10,000 | free-delivery threshold | 0 |

### Step 4: write the parameterized test

```java
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertThrows;

import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;
import org.junit.jupiter.params.provider.ValueSource;

class DeliveryFeeCalculatorTest {

    private final DeliveryFeeCalculator calculator =
            new DeliveryFeeCalculator();

    @ParameterizedTest(
            name = "order total {0} results in delivery fee {1}"
    )
    @CsvSource({
            "0,      700",
            "4999,   700",
            "5000,   300",
            "9999,   300",
            "10000,  0"
    })
    void shouldCalculateDeliveryFee(
            int orderTotal,
            int expectedFee
    ) {
        int actualFee = calculator.calculate(orderTotal);

        assertEquals(expectedFee, actualFee);
    }

    @ParameterizedTest
    @ValueSource(ints = {-1, -100, Integer.MIN_VALUE})
    void shouldRejectNegativeOrderTotal(int orderTotal) {
        assertThrows(
                IllegalArgumentException.class,
                () -> calculator.calculate(orderTotal)
        );
    }
}
```

The valid-fee rule and invalid-input rule remain separate. Each parameterized method has one clear purpose.

---

## 16. Practice

The following class calculates a score category:

```java
public final class ScoreClassifier {

    public String classify(int score) {
        if (score < 0 || score > 100) {
            throw new IllegalArgumentException(
                    "score must be between 0 and 100"
            );
        }
        if (score >= 90) {
            return "EXCELLENT";
        }
        if (score >= 75) {
            return "GOOD";
        }
        if (score >= 60) {
            return "PASS";
        }
        return "FAIL";
    }
}
```

Complete these tasks:

1. Write one `@CsvSource` test for representative values and boundaries of all four categories.
2. Give its invocations a readable display-name pattern.
3. Write one `@ValueSource` test verifying that invalid scores throw `IllegalArgumentException`.
4. Explain why valid and invalid cases should not be combined into one parameterized method.
5. State which source you would choose if every row also needed a complex `StudentProfile` object.

### Suggested solution

```java
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertThrows;

import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;
import org.junit.jupiter.params.provider.ValueSource;

class ScoreClassifierTest {

    private final ScoreClassifier classifier = new ScoreClassifier();

    @ParameterizedTest(name = "score {0} is classified as {1}")
    @CsvSource({
            "0,   FAIL",
            "59,  FAIL",
            "60,  PASS",
            "74,  PASS",
            "75,  GOOD",
            "89,  GOOD",
            "90,  EXCELLENT",
            "100, EXCELLENT"
    })
    void shouldClassifyValidScore(int score, String expectedCategory) {
        String actualCategory = classifier.classify(score);

        assertEquals(expectedCategory, actualCategory);
    }

    @ParameterizedTest(name = "score {0} is outside the valid range")
    @ValueSource(ints = {-1, 101, Integer.MIN_VALUE, Integer.MAX_VALUE})
    void shouldRejectScoreOutsideValidRange(int score) {
        assertThrows(
                IllegalArgumentException.class,
                () -> classifier.classify(score)
        );
    }
}
```

The valid cases return a category, while invalid cases throw an exception. They have different assertion structures and represent different rules, so separate methods communicate them more clearly.

If each row required a complex `StudentProfile`, `@MethodSource` would normally be the clearest option because the source method can construct typed domain objects.

---

## 17. Knowledge check

1. What does `@ParameterizedTest` change compared with `@Test`?
2. What is an invocation?
3. Why must every parameterized test declare an argument source?
4. When is `@ValueSource` appropriate?
5. Can `@ValueSource` provide two arguments for one invocation?
6. What is the difference between `@NullSource` and `@EmptySource` for a `String` parameter?
7. Does `@EmptySource` supply whitespace-only strings?
8. When is `@CsvSource` a good choice?
9. How are CSV columns mapped to method parameters?
10. How do you represent a CSV value that contains a comma?
11. What do an unquoted empty CSV column and `''` mean?
12. When should `@MethodSource` be preferred to `@CsvSource`?
13. Must a local `@MethodSource` factory be static?
14. What does `@EnumSource(PaymentStatus.class)` supply?
15. What is basic implicit argument conversion?
16. What do `{index}` and `{0}` mean in an invocation display-name pattern?
17. Does `@BeforeEach` run once for the complete parameterized method or before each invocation?
18. What is a strong sign that several rows should become separate test methods?
19. Why is it dangerous to calculate expected values by calling the production code being tested?
20. Which source would you choose for several `LocalDate`, domain-object, and expected-result combinations?

### Answers

1. It tells Jupiter to invoke the method repeatedly with argument sets supplied by one or more sources.
2. An invocation is one execution of the parameterized method with one argument set, reported as an individual test case.
3. Without a source, Jupiter has no argument sets with which to call the method.
4. Use it for one simple literal argument per invocation.
5. No. It supplies one argument per invocation; use `@CsvSource` or `@MethodSource` for several arguments.
6. `@NullSource` supplies `null`; `@EmptySource` supplies `""` for a `String`.
7. No. Combine it with `@ValueSource` to provide representative whitespace-only strings.
8. Use it for a compact table of several simple values per invocation.
9. They map by position: first column to first parameter, second column to second parameter, and so on.
10. Quote the field with single quotes inside the CSV record, or configure/use a clearer alternative representation.
11. An unquoted empty column represents `null`; `''` represents an empty string.
12. Prefer it for domain objects, programmatically built data, or data that is awkward or unsafe to express as CSV text.
13. Under the default `PER_METHOD` lifecycle, yes. It may be non-static with `PER_CLASS`; an external source factory is always static.
14. It supplies every constant of the `PaymentStatus` enum, one per invocation.
15. Jupiter converts a supported supplied value to the compatible type declared by the test parameter, such as CSV text to `int`, `LocalDate`, or an enum.
16. `{index}` is the one-based invocation number; `{0}` is the first argument.
17. It runs before each invocation.
18. The test body needs branches, different actions, different arrangements, or different assertion structures for different rows.
19. The expectation can reproduce the same defect as the production code, allowing incorrect behavior to pass.
20. `@MethodSource`, because it can construct and supply typed objects and several arguments.

---

## 18. Key takeaways

- A parameterized test runs one test method several times with different arguments.
- Each argument set creates a separately reported invocation.
- Parameterize one behavioral scenario, not merely several methods from the same feature.
- Use `@ValueSource` for one simple literal argument.
- Use `@NullSource`, `@EmptySource`, or `@NullAndEmptySource` for common missing-input boundaries.
- Use `@CsvSource` for a compact table of simple columns.
- Use `@MethodSource` for objects and programmatically created argument sets.
- Use `@EnumSource` for typed enum constants.
- Source arguments must match method parameters by number, position, and compatible type.
- Basic implicit conversion makes text-based sources usable with typed parameters.
- Display names should help a reader identify the failing data set.
- `@BeforeEach` and `@AfterEach` run for every invocation.
- Separate tests are clearer when scenarios require different actions, setup, or assertions.
- More rows do not automatically mean better coverage; choose representative cases and boundaries.

---

## 19. JUnit 5 and JUnit 6 compatibility note

The core parameterized-test APIs covered in this topic use stable Jupiter packages:

- `org.junit.jupiter.params.ParameterizedTest`;
- `org.junit.jupiter.params.provider.ValueSource`;
- `org.junit.jupiter.params.provider.CsvSource`;
- `org.junit.jupiter.params.provider.MethodSource`;
- `org.junit.jupiter.params.provider.EnumSource`;
- the null and empty source annotations.

The examples therefore have the same basic structure in modern JUnit 5 and JUnit 6 projects.

For this course:

- examples target **Java 25 and JUnit 6.1.2**;
- JUnit 6 requires Java 17 or later;
- a project restricted to Java 8–16 needs a compatible JUnit 5 version;
- JUnit 6 quotes text arguments in parameterized invocation display names by default, so reports may look slightly different from JUnit 5 reports;
- `@ParameterizedClass` is an experimental feature in current JUnit versions and is intentionally outside this topic, which focuses on parameterized test methods.

The dependency version may change over time. Follow the project's dependency-management policy instead of copying a version without checking it.

---

## Further reading

- [JUnit 6 User Guide: Parameterized Classes and Tests](https://docs.junit.org/6.1.2/writing-tests/parameterized-classes-and-tests.html)
- [JUnit 6 User Guide: Test Instance Lifecycle](https://docs.junit.org/6.1.2/writing-tests/test-instance-lifecycle.html)
- [JUnit 6 User Guide: Display Names](https://docs.junit.org/6.1.2/writing-tests/display-names.html)
- [JUnit 5.14 User Guide: Parameterized Classes and Tests](https://docs.junit.org/5.14.4/writing-tests/parameterized-classes-and-tests.html)