# Writing and Running Tests with JUnit Jupiter

## Learning objectives

By the end of this topic, you will be able to:

- explain the roles of the JUnit Platform and JUnit Jupiter;
- add JUnit Jupiter to a Maven or Gradle project;
- place production and test code in the standard project directories;
- write a test class and a test method using `@Test`;
- organize a simple test using Arrange–Act–Assert;
- choose and use common JUnit assertions;
- verify returned values, object state, and exceptional behavior;
- name test classes and methods clearly;
- run one test, one test class, or the complete test suite in an IDE;
- run tests with Maven or Gradle and interpret the result.

---

## 1. From a test case to executable test code

In the previous topic, we selected test cases from expected software behavior. A test case such as the following is useful even before any test code exists:

| Scenario | Input | Expected behavior |
| --- | --- | --- |
| Order reaches the free-delivery threshold | order total `5_000`, non-premium customer | delivery fee is `0` |

JUnit Jupiter lets us express this case as executable Java code:

```java
import static org.junit.jupiter.api.Assertions.assertEquals;

import org.junit.jupiter.api.Test;

class DeliveryFeeCalculatorTest {

    @Test
    void shouldProvideFreeDeliveryWhenOrderReachesThreshold() {
        DeliveryFeeCalculator calculator = new DeliveryFeeCalculator();

        int actualFee = calculator.calculate(5_000, false);

        assertEquals(0, actualFee);
    }
}
```

This test contains four essential elements:

1. the test class, `DeliveryFeeCalculatorTest`;
2. the test method, `shouldProvideFreeDeliveryWhenOrderReachesThreshold()`;
3. the `@Test` annotation, which marks the method as a test;
4. the assertion, which compares expected and actual behavior.

When JUnit runs the method, the test passes if `actualFee` is `0`. If it is not, the assertion fails and JUnit reports the difference.

> A test is not successful merely because its code executes. It must verify the relevant result or effect.

---

## 2. How modern JUnit is structured

JUnit is not one indivisible library. Modern JUnit consists of several related parts.

### 2.1 JUnit Platform

The **JUnit Platform** is the foundation for discovering and launching tests on the JVM. IDEs and build tools use the platform to run tests through registered test engines.

Think of it as the execution infrastructure:

- it receives a request to run tests;
- it discovers available test engines;
- it asks the appropriate engine to find and execute its tests;
- it reports the results to the IDE or build tool.

### 2.2 JUnit Jupiter

**JUnit Jupiter** provides the programming model used to write modern JUnit tests and the test engine that executes those tests.

It provides familiar APIs such as:

- `@Test`;
- `@BeforeEach` and other lifecycle annotations;
- `Assertions`;
- parameterized-test support;
- the Jupiter test engine.

The main imports used in this module begin with:

```java
org.junit.jupiter
```

The name `jupiter` remains in the packages in both JUnit 5 and JUnit 6.

### 2.3 JUnit Vintage

**JUnit Vintage** is an engine for running older JUnit 3 and JUnit 4 tests on the JUnit Platform. It is intended mainly for migration of legacy test suites and is deprecated in JUnit 6.

New tests in this course use JUnit Jupiter. They do not need the Vintage engine.

### 2.4 The execution path

When an IDE or build tool runs a Jupiter test, the simplified flow is:

```text
IDE or build tool -> JUnit Platform -> Jupiter test engine -> test methods
```

This explains why writing and running a test involve related but different pieces:

- the Jupiter API supplies `@Test` and assertions for compiling the test;
- the Jupiter engine discovers and executes the test;
- the JUnit Platform connects the engine to the IDE or build tool.

The `junit-jupiter` dependency used later in this topic is a convenient aggregator that brings in the Jupiter API, parameterized-test support, and the Jupiter engine.

---

## 3. Standard project structure

In a conventional Maven or Gradle Java project, production code and test code are separated:

```text
project-root/
|-- pom.xml
`-- src/
    |-- main/
    |   `-- java/
    |       `-- com/example/delivery/
    |           `-- DeliveryFeeCalculator.java
    `-- test/
        `-- java/
            `-- com/example/delivery/
                `-- DeliveryFeeCalculatorTest.java
```

- `src/main/java` contains production source code.
- `src/test/java` contains test source code.

The test class usually uses the same package as the production class it tests:

```java
package com.example.delivery;
```

This is a useful convention because it keeps the corresponding files easy to find. It also allows a test to access package-private members when that is intentionally part of the package design. It does not mean that tests should expose or directly verify private implementation details.

Test code is compiled separately and is not normally packaged as part of the application artifact. Dependencies with Maven's `test` scope or Gradle's `testImplementation` configuration are therefore available to tests without becoming normal application dependencies.

---

## 4. Adding JUnit to a project

The examples in this course target Java 25 and JUnit 6.1.2. The test source code also works with modern JUnit 5 versions unless a note says otherwise.

### 4.1 Maven configuration

A minimal Maven configuration can look like this:

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>delivery-calculator</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.release>25</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <junit.version>6.1.2</junit.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>${junit.version}</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.15.0</version>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.5.5</version>
            </plugin>
        </plugins>
    </build>
</project>
```

Important details are:

- `junit-jupiter` supplies the Jupiter modules needed to write and run the tests;
- `<scope>test</scope>` limits this dependency to test compilation and execution;
- Maven Compiler compiles the production and test sources for the configured Java release;
- Maven Surefire discovers and runs tests during the Maven `test` phase;
- the Java release and JUnit version are declared once as properties instead of being repeated.

JUnit 6 requires Java 17 or later at runtime. Java 25 therefore satisfies this requirement.

In a project managed by a framework such as Spring Boot, dependency versions may already be managed. Do not add or override versions mechanically without first checking the project's existing dependency management.

### 4.2 Gradle configuration

Maven is the primary build tool in this course. The equivalent basic Gradle configuration is:

```groovy
plugins {
    id 'java'
}

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(25)
    }
}

repositories {
    mavenCentral()
}

dependencies {
    testImplementation 'org.junit.jupiter:junit-jupiter:6.1.2'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}

test {
    useJUnitPlatform()
}
```

`useJUnitPlatform()` tells Gradle to run tests on the JUnit Platform. The launcher runtime dependency helps ensure that the build and IDE use a compatible platform launcher version.

---

## 5. Production code for the examples

The main examples test the following class from `src/main/java`:

```java
package com.example.delivery;

public final class DeliveryFeeCalculator {

    private static final int FREE_DELIVERY_THRESHOLD = 5_000;
    private static final int STANDARD_FEE = 500;

    public int calculate(int orderTotal, boolean premiumCustomer) {
        if (orderTotal < 0) {
            throw new IllegalArgumentException(
                    "Order total must not be negative: " + orderTotal
            );
        }

        if (premiumCustomer || orderTotal >= FREE_DELIVERY_THRESHOLD) {
            return 0;
        }

        return STANDARD_FEE;
    }
}
```

Its observable contract contains several behaviors:

- an ordinary small order has a delivery fee of `500`;
- an order of at least `5_000` has free delivery;
- a premium customer has free delivery;
- a negative order total is rejected with `IllegalArgumentException`.

Each behavior can become a focused test case.

---

## 6. Anatomy of a test class

Here is a complete initial test class from `src/test/java`:

```java
package com.example.delivery;

import static org.junit.jupiter.api.Assertions.assertEquals;

import org.junit.jupiter.api.Test;

class DeliveryFeeCalculatorTest {

    @Test
    void shouldChargeStandardFeeForOrdinarySmallOrder() {
        // Arrange
        DeliveryFeeCalculator calculator = new DeliveryFeeCalculator();

        // Act
        int actualFee = calculator.calculate(2_000, false);

        // Assert
        assertEquals(500, actualFee);
    }
}
```

### 6.1 The test class

A Jupiter test class is an ordinary Java class. It does not have to extend a JUnit base class.

Top-level test classes must not be `private`. They are commonly package-private, as in the example:

```java
class DeliveryFeeCalculatorTest {
```

Making a test class `public` is allowed, but it is not normally necessary.

### 6.2 The `@Test` annotation

The annotation:

```java
@Test
```

marks the following method as a Jupiter test method. It must be imported from:

```java
org.junit.jupiter.api.Test
```

This import is important. `org.junit.Test` belongs to JUnit 4 and should not be used for a new Jupiter test.

A basic Jupiter test method:

- is an instance method;
- must not be `private`;
- must not be `abstract`;
- must not return a value;
- usually has no parameters unless a JUnit feature or extension supplies them.

Package-private methods are a common style:

```java
@Test
void shouldChargeStandardFeeForOrdinarySmallOrder() {
    // test code
}
```

The method name does not need to begin with `test`. The annotation identifies it as a test.

### 6.3 Static imports for assertions

JUnit assertions are static methods of `org.junit.jupiter.api.Assertions`. A static import makes calls concise:

```java
import static org.junit.jupiter.api.Assertions.assertEquals;
```

Then the test can use:

```java
assertEquals(500, actualFee);
```

instead of:

```java
Assertions.assertEquals(500, actualFee);
```

Both forms are correct. Static imports are conventional in JUnit test code because they keep the verification easy to read.

---

## 7. Arrange–Act–Assert

A simple test is often easiest to understand when it follows three phases.

### Arrange

Prepare the object under test, its input, and any required initial state:

```java
DeliveryFeeCalculator calculator = new DeliveryFeeCalculator();
```

### Act

Execute the behavior being tested:

```java
int actualFee = calculator.calculate(2_000, false);
```

### Assert

Compare the actual behavior with the expected behavior:

```java
assertEquals(500, actualFee);
```

Together, these phases are called **Arrange–Act–Assert**, or **AAA**.

Blank lines are often enough to show the phases. Comments such as `// Arrange` can help beginners or make a complicated test easier to scan, but they do not need to be repeated mechanically in every small test.

The Act phase should normally contain one principal operation. This makes it clear which action produced the behavior being verified. Setup may call other methods when necessary, but the test should still have one identifiable subject.

AAA is introduced here as a practical way to structure a first test. Its role as a test-quality principle will be examined more deeply in the final topic of this module.

---

## 8. Naming tests clearly

JUnit does not require one universal naming convention. The useful rule is that a name should reveal the behavior and relevant condition.

### 8.1 Test class names

A common pattern is:

```text
ProductionClassName + Test
```

Examples:

- `DeliveryFeeCalculatorTest`;
- `PasswordValidatorTest`;
- `ShoppingCartTest`.

The `*Test` suffix is also recognized by Maven Surefire's default test-class discovery patterns.

### 8.2 Test method names

Compare these names:

```java
void testCalculate()
void calculateTest1()
void shouldProvideFreeDeliveryWhenOrderReachesThreshold()
```

The third name communicates the scenario and expected behavior. If it fails, the report already gives useful context.

Other consistent styles are possible:

```java
void calculate_returnsZero_whenCustomerIsPremium()
void givenPremiumCustomer_whenCalculatingFee_thenReturnsZero()
```

Choose a readable convention and apply it consistently. Do not make names longer than the behavior requires, but avoid names such as `test1`, `works`, or `calculate` that hide the purpose of the test.

### 8.3 `@DisplayName`

JUnit also supports custom display names:

```java
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

@Test
@DisplayName("Premium customers receive free delivery")
void premiumCustomerHasNoDeliveryFee() {
    // test code
}
```

`@DisplayName` changes the human-readable label in the test report. It does not replace the Java method name, and it is optional. Descriptive method names are sufficient for most basic tests.

---

## 9. Assertions: making a test self-validating

An **assertion** checks an expected condition. If the condition is satisfied, execution continues. If it is not, the assertion throws an `AssertionError`, and JUnit reports the test as failed.

All Jupiter assertions are static methods in:

```java
org.junit.jupiter.api.Assertions
```

The following sections cover a practical core set rather than every available assertion.

### 9.1 `assertEquals()` and `assertNotEquals()`

Use `assertEquals()` when the actual value should be equal to an expected value:

```java
assertEquals(500, actualFee);
```

The conventional parameter order is:

```java
assertEquals(expected, actual);
```

Keeping this order produces clearer failure reports:

```text
expected: <500> but was: <0>
```

For objects, `assertEquals()` uses equality semantics rather than requiring both references to point to the same object. In practice, the compared type needs an appropriate `equals()` implementation.

Use `assertNotEquals()` when two values must not be equal:

```java
assertNotEquals(500, freeDeliveryFee);
```

For floating-point calculations, use an overload with a tolerance, often called delta:

```java
assertEquals(0.3, 0.1 + 0.2, 0.000_001);
```

Exact equality is often unsafe for calculated `double` and `float` values because many decimal fractions cannot be represented exactly in binary floating-point form.

### 9.2 `assertTrue()` and `assertFalse()`

Use these assertions for boolean conditions:

```java
assertTrue(passwordValidator.isValid("correct-password"));
assertFalse(passwordValidator.isValid("short"));
```

Prefer the most direct expression. For example:

```java
assertTrue(result.isSuccessful());
```

is usually easier to understand than:

```java
assertEquals(true, result.isSuccessful());
```

### 9.3 `assertNull()` and `assertNotNull()`

Use these when `null` itself is part of the contract:

```java
assertNull(cache.find("missing-key"));
assertNotNull(customer.getId());
```

Do not add `assertNotNull()` automatically before every other assertion. If the next assertion already gives a clear failure and `null` is not a separate requirement, the extra assertion may add noise.

### 9.4 `assertSame()` and `assertNotSame()`

These assertions compare object identity:

```java
assertSame(original, returnedObject);
assertNotSame(original, copy);
```

- `assertEquals()` asks whether two values are logically equal.
- `assertSame()` asks whether two references point to the exact same object.

Use identity assertions only when identity is part of the expected behavior.

### 9.5 `assertArrayEquals()`

Java arrays do not implement content-based `equals()`. Use `assertArrayEquals()` to compare their elements:

```java
int[] expected = {2, 4, 6};
int[] actual = numberService.evenNumbersUpTo(6);

assertArrayEquals(expected, actual);
```

Using `assertEquals(expected, actual)` here would compare array objects rather than their contents.

For collections such as `List`, `assertEquals()` normally works because collection implementations define content-based equality. Order is significant for lists but not normally for sets.

### 9.6 Optional failure messages

Most assertion methods accept a failure message as the last argument:

```java
assertEquals(
        0,
        actualFee,
        "An order at the threshold should have free delivery"
);
```

The message is displayed only when the assertion fails. Use it when it adds information that is not already clear from the test name and values.

For an expensive message, a `Supplier<String>` can build it lazily:

```java
assertEquals(expected, actual, () -> "Unexpected fee for order " + orderId);
```

Do not write a message that merely repeats “expected equals actual.” A descriptive test name and JUnit's expected/actual output are often already sufficient.

---

## 10. Testing exceptional behavior with `assertThrows()`

Throwing an exception can be correct behavior. A negative order total, for example, should be rejected.

Use `assertThrows()` to verify the exception type:

```java
import static org.junit.jupiter.api.Assertions.assertThrows;

@Test
void shouldRejectNegativeOrderTotal() {
    DeliveryFeeCalculator calculator = new DeliveryFeeCalculator();

    assertThrows(
            IllegalArgumentException.class,
            () -> calculator.calculate(-1, false)
    );
}
```

The second argument is an `Executable`, expressed here as a lambda. JUnit invokes that lambda and observes what it throws.

The test fails if:

- no exception is thrown; or
- the thrown exception is not compatible with the expected type.

`assertThrows()` returns the captured exception, so the test can verify its relevant data:

```java
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertThrows;

@Test
void shouldExplainWhyNegativeOrderTotalIsRejected() {
    DeliveryFeeCalculator calculator = new DeliveryFeeCalculator();

    IllegalArgumentException exception = assertThrows(
            IllegalArgumentException.class,
            () -> calculator.calculate(-1, false)
    );

    assertEquals(
            "Order total must not be negative: -1",
            exception.getMessage()
    );
}
```

Only the action expected to throw should be inside the lambda. This version is too broad:

```java
assertThrows(IllegalArgumentException.class, () -> {
    TestData data = TestData.from("invalid");
    calculator.calculate(data.orderTotal(), data.premiumCustomer());
});
```

If `TestData.from()` throws the exception, the assertion can pass without ever testing the calculator. Prepare the input before `assertThrows()` whenever possible.

### Expected type and subtype

`assertThrows(IllegalArgumentException.class, ...)` also succeeds if the code throws a subclass of `IllegalArgumentException`.

When the exact runtime type is part of the contract, Jupiter also provides:

```java
assertThrowsExactly(ExpectedException.class, executable);
```

For most tests, expecting the meaningful public exception type with `assertThrows()` is sufficient.

### Avoid manual `try`/`catch`

This approach is error-prone:

```java
try {
    calculator.calculate(-1, false);
} catch (IllegalArgumentException exception) {
    assertEquals("Order total must not be negative: -1", exception.getMessage());
}
```

If no exception is thrown, the test reaches the end and passes. `assertThrows()` handles the absence of the exception correctly and expresses the expectation more clearly.

---

## 11. Verifying that code does not throw

Normally, a test does not need a special assertion for the absence of exceptions. If the Act phase unexpectedly throws, the test already fails.

Use `assertDoesNotThrow()` when “this operation must complete without throwing” is itself the behavior you want to emphasize or when you want to capture a returned value:

```java
int fee = assertDoesNotThrow(
        () -> calculator.calculate(2_000, false)
);

assertEquals(500, fee);
```

Do not wrap every normal action in `assertDoesNotThrow()`. It can make simple tests more verbose without improving their meaning.

---

## 12. Grouping related checks with `assertAll()`

Suppose a method returns a record containing several related parts of one result:

```java
public record PriceBreakdown(int subtotal, int discount, int total) {
}
```

A test may need to check all three parts:

```java
import static org.junit.jupiter.api.Assertions.assertAll;
import static org.junit.jupiter.api.Assertions.assertEquals;

@Test
void shouldApplyPremiumDiscountToPriceBreakdown() {
    PriceCalculator calculator = new PriceCalculator();

    PriceBreakdown result = calculator.calculate(1_000, true);

    assertAll(
            "price breakdown",
            () -> assertEquals(1_000, result.subtotal()),
            () -> assertEquals(100, result.discount()),
            () -> assertEquals(900, result.total())
    );
}
```

Without `assertAll()`, JUnit stops the test method at the first failed assertion. With `assertAll()`, JUnit executes all assertions in the group and reports all their failures together.

Use `assertAll()` for several related properties of one result or one coherent scenario. Do not use it to combine unrelated test cases:

```java
// Poor organization: three different business scenarios in one test
assertAll(
        () -> assertEquals(500, calculator.calculate(2_000, false)),
        () -> assertEquals(0, calculator.calculate(5_000, false)),
        () -> assertEquals(0, calculator.calculate(100, true))
);
```

Separate test methods, or later a parameterized test, describe those independent scenarios more clearly.

---

## 13. A complete basic test class

The calculator's main rules can be tested as follows:

```java
package com.example.delivery;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertThrows;

import org.junit.jupiter.api.Test;

class DeliveryFeeCalculatorTest {

    @Test
    void shouldChargeStandardFeeForOrdinarySmallOrder() {
        DeliveryFeeCalculator calculator = new DeliveryFeeCalculator();

        int actualFee = calculator.calculate(2_000, false);

        assertEquals(500, actualFee);
    }

    @Test
    void shouldProvideFreeDeliveryWhenOrderReachesThreshold() {
        DeliveryFeeCalculator calculator = new DeliveryFeeCalculator();

        int actualFee = calculator.calculate(5_000, false);

        assertEquals(0, actualFee);
    }

    @Test
    void shouldProvideFreeDeliveryToPremiumCustomer() {
        DeliveryFeeCalculator calculator = new DeliveryFeeCalculator();

        int actualFee = calculator.calculate(100, true);

        assertEquals(0, actualFee);
    }

    @Test
    void shouldRejectNegativeOrderTotal() {
        DeliveryFeeCalculator calculator = new DeliveryFeeCalculator();

        IllegalArgumentException exception = assertThrows(
                IllegalArgumentException.class,
                () -> calculator.calculate(-1, false)
        );

        assertEquals(
                "Order total must not be negative: -1",
                exception.getMessage()
        );
    }
}
```

There is some repeated setup here. That is acceptable in a small first example. The next topic will show when and how lifecycle hooks can provide common setup without creating shared-state problems.

---

## 14. How tests are discovered and run

Running tests involves two main activities:

1. **Discovery:** the runner finds test classes and test methods.
2. **Execution:** the appropriate test engine invokes them and records their results.

The exact selection depends on what you run:

- one test method;
- one test class;
- a package or directory;
- every test in the module or project.

### Test-class names in Maven

By default, Maven Surefire scans for common test class patterns, including:

```text
Test*.java
*Test.java
*Tests.java
*TestCase.java
```

This is another reason why `DeliveryFeeCalculatorTest` is a practical name. A class named `DeliveryFeeCalculatorCheck` may run from an IDE when selected directly but be ignored by a default Maven build.

The `@Test` annotation identifies test methods inside a discovered class. A method without `@Test` is an ordinary helper method and will not be run as a test merely because its name starts with `test`.

---

## 15. Running tests in IntelliJ IDEA

IntelliJ IDEA can run JUnit Platform tests directly.

### Run one test method

Open the test class and use the run icon in the gutter beside the test method. Choose **Run** for that method.

This is useful when developing or debugging one scenario.

### Run one test class

Use the run icon beside the class declaration. IDEA runs all test methods in that class.

### Run a package, source root, or module

In the Project view, right-click the relevant package, `src/test/java` directory, module, or project and choose the corresponding **Run Tests** action.

This is useful before committing a change because a local edit can affect tests beyond the class currently open.

### Rerun and debug

The test runner window lets you:

- rerun the same selection;
- run only failed tests;
- navigate from a failure to the relevant source line;
- inspect expected and actual values;
- start the test under the debugger.

The IDE is convenient, but the build-tool command is still important. Continuous integration normally runs the build rather than reproducing an individual developer's IDE configuration.

---

## 16. Running tests with Maven

Run all tests discovered by Maven Surefire:

```bash
mvn test
```

Run one test class:

```bash
mvn -Dtest=DeliveryFeeCalculatorTest test
```

Run one test method:

```bash
mvn -Dtest=DeliveryFeeCalculatorTest#shouldRejectNegativeOrderTotal test
```

Maven performs the earlier lifecycle phases as needed, including compilation of production and test source code, and then executes the tests in the `test` phase.

A successful summary may look similar to:

```text
Tests run: 4, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

A failing assertion causes a failure report and normally ends the build with:

```text
BUILD FAILURE
```

The exact formatting can vary by tool and plugin version. Focus on the selected test, the failure message, expected and actual values, and the first relevant line of your own code in the stack trace.

---

## 17. Running tests with Gradle

On macOS or Linux, run the complete test suite with the Gradle Wrapper:

```bash
./gradlew test
```

On Windows:

```text
gradlew.bat test
```

Run one test class:

```bash
./gradlew test --tests com.example.delivery.DeliveryFeeCalculatorTest
```

Run one test method:

```bash
./gradlew test --tests "com.example.delivery.DeliveryFeeCalculatorTest.shouldRejectNegativeOrderTotal"
```

Prefer the project's wrapper, `mvnw` or `gradlew`, when it is provided. The wrapper helps the team and continuous integration use the build-tool version configured for the project.

---

## 18. Interpreting test results

### Passed or successful

The test completed and all its assertions were satisfied.

This means only that the tested expectation held for that execution. It does not prove that every behavior is correct.

### Failed

The test ran but did not satisfy an assertion, or it ended with an unexpected exception.

For an assertion failure, inspect:

- the test name;
- the expected value;
- the actual value;
- the assertion's failure message;
- the line where the assertion failed.

A failing test does not automatically mean production code is wrong. The test itself, its data, or its expectation may be incorrect. Compare both sides with the requirement before changing code.

### Disabled, skipped, or aborted

Reports use slightly different terminology:

- a **disabled** test is deliberately excluded, for example with `@Disabled`;
- an **aborted** test started but was stopped because an assumption was not satisfied;
- build tools may group disabled and aborted executions under a broader **skipped** count.

A skipped test has not demonstrated that its expected behavior works. Do not treat “not failed” as equivalent to “passed.”

### Compilation error

If production or test code does not compile, test execution may never begin. A missing import, incompatible method call, or syntax error is a build problem, not a failed assertion.

### No tests found

If the report says that no tests were found, check:

- whether the test is under `src/test/java`;
- whether `@Test` comes from `org.junit.jupiter.api`;
- whether the JUnit dependency and engine are present;
- whether the build tool is configured for the JUnit Platform;
- whether the class name matches the build tool's discovery patterns;
- whether the correct module or task was selected.

---

## 19. Common mistakes

### 19.1 Importing the JUnit 4 annotation

Incorrect for a new Jupiter test:

```java
import org.junit.Test;
```

Correct:

```java
import org.junit.jupiter.api.Test;
```

IDE auto-import can choose the wrong class when several versions are on the classpath. Always check the complete package.

### 19.2 Writing a method without `@Test`

This method will not be discovered as a Jupiter test:

```java
void shouldCalculateFee() {
    assertEquals(500, calculator.calculate(2_000, false));
}
```

The method name alone does not make it a test.

### 19.3 Reversing expected and actual values

Misleading:

```java
assertEquals(actualFee, 500);
```

Conventional:

```java
assertEquals(500, actualFee);
```

The comparison may behave the same, but the failure report becomes harder to interpret when the roles are reversed.

### 19.4 Printing instead of asserting

This is manual observation, not automated verification:

```java
System.out.println(calculator.calculate(2_000, false));
```

Replace it with an assertion that defines the expected result.

### 19.5 Comparing array contents with `assertEquals()`

Use `assertArrayEquals()` for arrays. `assertEquals()` normally compares array identity rather than elements.

### 19.6 Expecting an exception that is too general

This assertion may hide many unrelated defects:

```java
assertThrows(Exception.class, () -> calculator.calculate(-1, false));
```

Expect the most meaningful public exception type promised by the contract:

```java
assertThrows(
        IllegalArgumentException.class,
        () -> calculator.calculate(-1, false)
);
```

### 19.7 Putting too much code inside `assertThrows()`

If setup and the operation under test are both inside the lambda, an exception from setup can make the test pass for the wrong reason. Narrow the lambda to the intended action.

### 19.8 Using `assertAll()` to combine unrelated scenarios

Grouped assertions are for related properties of one result. Independent business scenarios should normally be separate tests or inputs to one parameterized test.

### 19.9 Assuming a green IDE run guarantees a green build

The IDE and build tool can use different test selections or classpaths. Run the project's normal build command before committing, especially after changing dependencies or configuration.

---

## 20. Worked example: diagnose a failure

Suppose the threshold condition is accidentally changed to:

```java
if (premiumCustomer || orderTotal > FREE_DELIVERY_THRESHOLD) {
    return 0;
}
```

The boundary test runs:

```java
@Test
void shouldProvideFreeDeliveryWhenOrderReachesThreshold() {
    DeliveryFeeCalculator calculator = new DeliveryFeeCalculator();

    int actualFee = calculator.calculate(5_000, false);

    assertEquals(0, actualFee);
}
```

JUnit reports a difference similar to:

```text
expected: <0> but was: <500>
```

Read the failure systematically:

1. **Test name:** the behavior concerns reaching the threshold.
2. **Expected value:** the requirement says the fee should be `0`.
3. **Actual value:** the implementation returned `500`.
4. **Input:** the order total is exactly `5_000`.
5. **Production condition:** `>` excludes the boundary value.

The appropriate production condition is:

```java
orderTotal >= FREE_DELIVERY_THRESHOLD
```

Do not “fix” the test by changing its expected value to match the faulty implementation. The requirement determines the expectation.

---

## 21. Practice

Consider this production class:

```java
public final class TemperatureConverter {

    public double celsiusToFahrenheit(double celsius) {
        if (celsius < -273.15) {
            throw new IllegalArgumentException(
                    "Temperature cannot be below absolute zero"
            );
        }

        return celsius * 9 / 5 + 32;
    }
}
```

### Task 1: write a normal-case test

Write a Jupiter test verifying that `0` degrees Celsius is converted to `32` degrees Fahrenheit. Use an appropriate delta for comparing `double` values.

### Task 2: write a boundary test

Verify that `-273.15` is accepted and converted to `-459.67`, allowing a small floating-point delta.

### Task 3: write an exception test

Verify that `-273.16` causes `IllegalArgumentException` and check its message.

### Task 4: run the tests

Run:

1. the exception test method in the IDE;
2. the complete `TemperatureConverterTest` class in the IDE;
3. all project tests with Maven or Gradle.

### Suggested solution

```java
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertThrows;

import org.junit.jupiter.api.Test;

class TemperatureConverterTest {

    private static final double DELTA = 0.000_001;

    @Test
    void shouldConvertZeroCelsiusToThirtyTwoFahrenheit() {
        TemperatureConverter converter = new TemperatureConverter();

        double actual = converter.celsiusToFahrenheit(0);

        assertEquals(32.0, actual, DELTA);
    }

    @Test
    void shouldConvertAbsoluteZero() {
        TemperatureConverter converter = new TemperatureConverter();

        double actual = converter.celsiusToFahrenheit(-273.15);

        assertEquals(-459.67, actual, DELTA);
    }

    @Test
    void shouldRejectTemperatureBelowAbsoluteZero() {
        TemperatureConverter converter = new TemperatureConverter();

        IllegalArgumentException exception = assertThrows(
                IllegalArgumentException.class,
                () -> converter.celsiusToFahrenheit(-273.16)
        );

        assertEquals(
                "Temperature cannot be below absolute zero",
                exception.getMessage()
        );
    }
}
```

---

## 22. Knowledge check

1. What is the difference between the JUnit Platform and JUnit Jupiter?
2. Why is JUnit declared with Maven's `test` scope?
3. Where should production and test Java files normally be placed?
4. What makes a method a Jupiter test method?
5. Does a test method have to be `public` or begin with `test`?
6. What are the three parts of Arrange–Act–Assert?
7. What is the conventional argument order for `assertEquals()`?
8. When should `assertSame()` be used instead of `assertEquals()`?
9. Why is `assertArrayEquals()` needed for Java arrays?
10. What does `assertThrows()` return, and why is that useful?
11. Why should only the expected failing operation normally be placed inside the `assertThrows()` lambda?
12. What is an appropriate use of `assertAll()`?
13. What command runs the complete test suite with Maven?
14. Name two reasons why Maven might report that no tests were found.
15. Does a passed test prove that the entire application is correct?

### Answers

1. The JUnit Platform provides infrastructure for discovering and launching tests. JUnit Jupiter provides the modern JUnit programming model and an engine that runs Jupiter tests.
2. JUnit is needed to compile and run tests but should not normally become a production application dependency.
3. Production files normally go under `src/main/java`, and test files under `src/test/java`.
4. It is an eligible method annotated with `org.junit.jupiter.api.Test`.
5. No. Package-private test methods are common, and `@Test`, not a name prefix, identifies the method.
6. Arrange prepares the situation, Act executes the behavior, and Assert verifies the result or effect.
7. Expected value first, actual value second.
8. Use `assertSame()` only when the two references are expected to identify the exact same object.
9. Java arrays do not define content-based `equals()`; `assertArrayEquals()` compares their elements.
10. It returns the captured exception. The test can then verify its message or other relevant data.
11. A setup operation could throw the same exception and make the test pass without exercising the intended behavior.
12. Use it to verify several related properties of one result or one coherent scenario and report all failures together.
13. `mvn test`.
14. Possible reasons include a wrong source directory, a class name outside Surefire's default patterns, a missing Jupiter engine, a wrong `@Test` import, or missing JUnit Platform configuration.
15. No. It provides evidence only for its specific scenario and expectation.

---

## 23. Key takeaways

- The JUnit Platform launches tests, while JUnit Jupiter provides the modern programming model and test engine used in this course.
- Production code belongs in `src/main/java`; test code belongs in `src/test/java`.
- A Jupiter test is a Java method annotated with `org.junit.jupiter.api.Test`.
- A clear basic test separates Arrange, Act, and Assert.
- Descriptive test names communicate the condition and expected behavior.
- Assertions make tests self-validating; choose an assertion that matches the kind of result being checked.
- Use `assertThrows()` for expected exceptions and inspect the returned exception when its data matters.
- Use `assertAll()` for related properties of one result, not for unrelated scenarios.
- Run tests both in the IDE and through the project's build tool.
- A skipped test has not passed, and a compilation error can prevent tests from running at all.
- A failing test requires investigation of both the production code and the test expectation.

---

## 24. JUnit 5 and JUnit 6 compatibility note

JUnit 5 and JUnit 6 both use the JUnit Platform and the Jupiter programming model. The basic source code in this topic uses stable APIs from `org.junit.jupiter.*` and is generally the same in modern JUnit 5 and JUnit 6 projects.

For this course:

- examples target **Java 25 and JUnit 6.1.2**;
- JUnit 6 requires Java 17 or later;
- a project that must remain on Java 8–16 needs a compatible JUnit 5 version instead;
- JUnit 6 aligns the versions of Platform, Jupiter, and Vintage artifacts;
- Vintage is not needed for new Jupiter tests and is deprecated in JUnit 6.

The dependency version may change over time. In a real project, follow the project's dependency-management policy rather than copying a version without checking it.

---

## Further reading

- [JUnit 6 User Guide: Overview](https://docs.junit.org/6.1.2/overview.html)
- [JUnit 6 User Guide: Test Classes and Methods](https://docs.junit.org/6.1.2/writing-tests/test-classes-and-methods.html)
- [JUnit 6 User Guide: Annotations](https://docs.junit.org/6.1.2/writing-tests/annotations.html)
- [JUnit 6 User Guide: Assertions](https://docs.junit.org/6.1.2/writing-tests/assertions.html)
- [JUnit 6 User Guide: IDE Support](https://docs.junit.org/6.1.2/running-tests/ide-support.html)
- [JUnit 6 User Guide: Build Support](https://docs.junit.org/6.1.2/running-tests/build-support.html)
- [Apache Maven Compiler Plugin](https://maven.apache.org/plugins/maven-compiler-plugin/)
- [Apache Maven Surefire Plugin](https://maven.apache.org/surefire/maven-surefire-plugin/)