# Unit Testing Foundations: Purpose, Scope, and Test Cases

## Learning objectives

By the end of this topic, you will be able to:

- define a **unit** and a **unit test**;
- explain how unit tests provide fast feedback and make refactoring safer;
- explain how tests document software behavior through examples;
- describe the place of unit tests in the testing pyramid;
- distinguish unit tests from integration and end-to-end tests;
- identify what a good unit test should and should not cover;
- choose useful test cases for normal, boundary, invalid, and exceptional behavior.

---

## 1. Why test software?

A program can compile successfully and still behave incorrectly.

Consider a method that calculates a delivery fee:

```java
public final class DeliveryFeeCalculator {

    public int calculate(int orderTotal, boolean premiumCustomer) {
        if (orderTotal < 0) {
            throw new IllegalArgumentException("Order total must not be negative");
        }

        if (premiumCustomer || orderTotal >= 5_000) {
            return 0;
        }

        return 500;
    }
}
```

The compiler can verify that the code follows Java syntax and type rules. It cannot determine whether the business rules were implemented correctly.

For example:

- Should an order with a total of exactly `5_000` have free delivery?
- Should a premium customer receive free delivery for a small order?
- What should happen when the order total is negative?
- Is `500` the correct standard delivery fee?

Testing compares the actual behavior of software with its expected behavior.

> A test is evidence about a particular behavior under particular conditions. It is not proof that the entire program contains no defects.

Testing is necessary because even simple code can contain:

- misunderstood requirements;
- incorrect conditions;
- boundary errors;
- missing validation;
- unexpected interactions between components;
- regressions introduced by later changes.

### Manual checking and automated testing

A developer could run the application, enter several values, and inspect the results manually. This can be useful during exploration, but it is slow and difficult to repeat consistently.

An automated test:

1. prepares a known situation;
2. executes the code;
3. compares the actual result with the expected result;
4. reports success or failure automatically.

Because automated tests can be run repeatedly, they can become a permanent part of the development process.

---

## 2. What is a unit?

There is no single physical size that always defines a unit.

A unit is a small part of software behavior that can be checked independently enough to provide focused feedback. In object-oriented Java code, the unit is often:

- a method;
- the public behavior of a class;
- a small group of closely related classes working in memory.

For beginners, it is often practical to think of a unit as the behavior of one method or one class. However, this is a guideline rather than a universal rule.

### The unit is behavior, not a number of lines

The following method is short, but it contains several distinct behaviors:

```java
public int calculate(int orderTotal, boolean premiumCustomer)
```

Its behavior includes:

- returning the standard fee for an ordinary small order;
- returning zero for an order that reaches the free-delivery threshold;
- returning zero for a premium customer;
- rejecting a negative order total.

One method may therefore require several test cases.

Conversely, a useful unit does not always correspond to exactly one method. A public operation may internally call several private methods. Tests normally verify the result of the public operation instead of testing each private step separately.

---

## 3. What is a unit test?

A **unit test** is an automated test that checks a small unit of behavior in a controlled context and produces a clear pass-or-fail result.

A useful unit test usually has the following characteristics:

- **Focused:** it checks one behavior or one coherent scenario.
- **Automated:** it performs the verification without manual inspection.
- **Controlled:** relevant input and dependencies are known.
- **Repeatable:** the same conditions produce the same result.
- **Fast:** it normally runs without slow external infrastructure.
- **Diagnostic:** when it fails, the failure points to a relatively small area of the code.

### A first look at a unit test

The detailed JUnit syntax will be introduced in the next topic. For now, focus on the behavior described by the test:

```java
import static org.junit.jupiter.api.Assertions.assertEquals;

import org.junit.jupiter.api.Test;

class DeliveryFeeCalculatorTest {

    private final DeliveryFeeCalculator calculator =
            new DeliveryFeeCalculator();

    @Test
    void shouldProvideFreeDeliveryWhenOrderReachesThreshold() {
        int fee = calculator.calculate(5_000, false);

        assertEquals(0, fee);
    }
}
```

This test describes one rule:

> A non-premium customer receives free delivery when the order total is exactly `5_000`.

The test does not ask a person to print or inspect the value. The assertion compares the expected fee, `0`, with the actual result returned by the calculator.

### Unit test versus test case

The terms are closely related, but they emphasize different ideas:

- A **test case** describes a scenario: conditions, input, action, and expected result.
- A **unit test** is automated code that checks a unit of behavior.

One test method commonly implements one test case. A parameterized test, which will be introduced later, can execute the same test logic for several test cases.

---

## 4. Why unit tests are valuable

### 4.1 Fast feedback

Unit tests usually execute code directly in memory. They do not normally need a deployed application, a browser, a real network service, or a production database.

This makes them suitable for frequent execution:

- after implementing a small change;
- before committing code;
- during code review;
- in a continuous integration pipeline;
- while investigating a defect.

Fast feedback shortens the time between introducing a problem and discovering it. A defect found immediately is usually easier to locate than a defect discovered after many unrelated changes.

### 4.2 Safer refactoring

**Refactoring** changes the internal structure of code without intentionally changing its observable behavior.

For example, the original calculator could be refactored to extract a named constant and a helper method:

```java
public final class DeliveryFeeCalculator {

    private static final int FREE_DELIVERY_THRESHOLD = 5_000;
    private static final int STANDARD_FEE = 500;

    public int calculate(int orderTotal, boolean premiumCustomer) {
        validateOrderTotal(orderTotal);

        if (premiumCustomer || orderTotal >= FREE_DELIVERY_THRESHOLD) {
            return 0;
        }

        return STANDARD_FEE;
    }

    private void validateOrderTotal(int orderTotal) {
        if (orderTotal < 0) {
            throw new IllegalArgumentException("Order total must not be negative");
        }
    }
}
```

The public behavior should remain unchanged. A set of tests covering the important rules can detect an accidental behavioral change during the refactoring.

Tests do not make refactoring risk-free. Their protection is limited by the scenarios they cover and by the accuracy of their expectations. However, a relevant test suite gives developers much more confidence than manual memory alone.

### 4.3 Protection against regressions

A **regression** occurs when a previously working behavior stops working after a change.

Suppose a developer changes the condition from:

```java
orderTotal >= 5_000
```

to:

```java
orderTotal > 5_000
```

Most orders may still be handled correctly, but the boundary value `5_000` is now wrong. A test for that exact boundary will fail and reveal the regression.

### 4.4 Documentation by examples

Good tests show how the code is expected to behave in concrete situations.

Compare a general requirement:

> Large orders qualify for free delivery.

with executable examples:

- `4_999`, non-premium customer → delivery fee `500`;
- `5_000`, non-premium customer → delivery fee `0`;
- `100`, premium customer → delivery fee `0`;
- `-1` → `IllegalArgumentException`.

The examples clarify details that the general sentence leaves ambiguous.

Tests are especially useful as documentation when they have:

- descriptive names;
- readable input data;
- clear expected results;
- little irrelevant setup;
- assertions that describe the business rule being verified.

Tests are not a complete replacement for requirements, API documentation, or architectural documentation. They document selected examples of behavior, not the whole purpose and design of a system.

### 4.5 Design feedback

Code that is difficult to test is often difficult to use or maintain.

For example, a class that creates database connections, reads the system clock, calls a remote API, performs calculations, and formats output inside one method has too many responsibilities. Difficulty writing focused tests can reveal excessive coupling or unclear boundaries.

Testing does not automatically create good design, but it can expose design problems early.

---

## 5. Unit tests in the testing pyramid

Software is tested at different levels because no single type of test can answer every question efficiently.

The **testing pyramid** is a model for thinking about a balanced automated test suite. From its broad base to its narrow top, it is commonly presented as:

1. **Unit tests** — the broad base;
2. **integration tests** — the middle layer;
3. **end-to-end tests** — the narrow top.

The model suggests:

- many focused, fast tests at the unit level;
- fewer tests that verify interactions between components;
- a smaller number of broad tests that exercise complete user flows.

It is a guideline, not a mathematical rule. The exact balance depends on the architecture, risks, and type of system.

### 5.1 Unit tests

Unit tests answer focused questions such as:

- Does this pricing rule return the correct discount?
- Does this validator reject a blank value?
- Does this parser correctly interpret this input?
- Does this operation throw the required exception for invalid data?

They are normally fast and give precise feedback.

### 5.2 Integration tests

Integration tests check whether parts of the system work together correctly.

Examples:

- Does a repository correctly store and retrieve data from the configured database?
- Does JSON serialization match the expected format?
- Does an application component communicate correctly with a message broker?

They provide confidence about boundaries and configuration that a pure unit test cannot provide. They are usually slower and require more setup.

### 5.3 End-to-end tests

End-to-end tests exercise a broad workflow through much or all of the application.

Example:

> A customer signs in, adds an item to the cart, completes payment, and sees the created order.

These tests provide confidence that major parts work together from a user-visible perspective. However, they tend to be slower, more expensive to maintain, and less precise when they fail.

### Comparing the levels

| Question | Unit test | Integration test | End-to-end test |
| --- | --- | --- | --- |
| Typical scope | One focused unit of behavior | Interaction between components or infrastructure | Complete workflow |
| Execution speed | Usually very fast | Usually slower | Usually slowest |
| External infrastructure | Normally avoided or controlled | Often included intentionally | Commonly required |
| Failure diagnosis | Usually focused | May have several possible causes | Often broad |
| Main confidence | Local logic is correct | Boundaries work together | User workflow works |

The levels complement one another. A large number of unit tests cannot prove that database configuration or an HTTP endpoint works. A few end-to-end tests cannot efficiently describe every boundary case in a calculation.

---

## 6. What should a unit test cover?

A good unit test covers **observable behavior**: a result or effect that can be observed through the unit's public contract.

Observable behavior can include:

- a returned value;
- a thrown exception;
- a change in publicly observable state;
- an interaction with an external boundary when that interaction is part of the contract.

At this stage, examples will focus mainly on returned values and exceptions. Stubs, mocks, and interaction testing will be covered in the next module.

### Test business rules, not Java itself

Suppose a record is used only to hold a name and a price:

```java
public record Product(String name, int price) {
}
```

Testing that Java-generated accessors return the constructor values is rarely valuable if no custom behavior exists. Such a test mostly verifies a language feature.

In contrast, a custom validation or pricing rule is application behavior and deserves focused tests.

### Test through the public contract

Consider the refactored calculator with a private `validateOrderTotal` method. The private method is an implementation detail. It can be renamed, inlined, or replaced without changing the class contract.

The behavior should be tested through the public method:

```java
calculator.calculate(-1, false)
```

The test verifies that the public operation rejects invalid input. It does not need direct access to the private helper.

Testing private methods directly tends to:

- couple tests to internal implementation;
- make safe refactoring harder;
- duplicate checks already possible through public behavior;
- hide the possibility that the class has too many responsibilities.

If important logic can only be tested by directly accessing a large private method, consider whether that logic belongs in a separate class with its own public contract.

### One coherent behavior per test

A test should normally have one clear reason to fail.

This does not mean that every test may contain only one assertion. Several assertions can verify one coherent result. For example, a test for creating an order might check the order ID, status, and total if all three describe the same created order.

However, one test should not usually combine unrelated cases such as:

- free delivery;
- invalid input;
- premium-customer behavior;
- standard delivery.

Separate scenarios make failures easier to understand.

---

## 7. What should a unit test not cover?

### 7.1 Internal implementation details

A test should not fail merely because correct code was reorganized.

Fragile test idea:

> Verify that `calculate()` calls private method `validateOrderTotal()` exactly once.

Useful behavior:

> Verify that `calculate()` rejects a negative order total.

The first statement describes how the code currently works. The second describes what the code promises to do.

### 7.2 Unrelated behaviors in one test

A long test that verifies many business rules is hard to understand and diagnose. Split unrelated scenarios into focused tests.

### 7.3 Real infrastructure unless the integration is the subject

A test that requires a real database, network service, file system location, or deployed server is normally not a unit test. It may still be a valuable integration or end-to-end test.

The label matters less than understanding the trade-off: external infrastructure usually makes a test slower, harder to control, and more vulnerable to failures unrelated to the local logic.

### 7.4 Trivial framework or language behavior

Tests should add confidence about application behavior. Avoid tests whose only purpose is to prove that Java assignment, a standard collection, or a framework-generated accessor works as documented.

### 7.5 Every possible input

Most methods have too many possible inputs to test exhaustively. For an `int` parameter alone, there are billions of possible values.

Effective testing selects representative cases based on behavior and risk.

---

## 8. Choosing useful test cases

Before writing test code, identify the rules and divide the input space into meaningful groups.

For `DeliveryFeeCalculator`, the rules are:

1. A negative order total is invalid.
2. A premium customer receives free delivery.
3. A non-premium customer receives free delivery when the order total is at least `5_000`.
4. Other orders have a delivery fee of `500`.

From these rules, choose cases from several categories.

### 8.1 Happy-path cases

A **happy path** is a normal, valid use of the unit.

Examples:

| Input | Expected result | Rule demonstrated |
| --- | --- | --- |
| `orderTotal = 2_000`, `premiumCustomer = false` | `500` | Standard delivery |
| `orderTotal = 6_000`, `premiumCustomer = false` | `0` | Free delivery for a large order |
| `orderTotal = 100`, `premiumCustomer = true` | `0` | Premium benefit |

Testing only happy paths is not sufficient, but they confirm the main intended behavior.

### 8.2 Boundary cases

A **boundary** is a point where behavior changes.

The free-delivery threshold is `5_000`. Useful values are:

- immediately below the boundary: `4_999`;
- exactly at the boundary: `5_000`;
- immediately above the boundary: `5_001`.

Boundary cases are valuable because comparison operators are easy to implement incorrectly:

- `<` instead of `<=`;
- `>` instead of `>=`;
- an incorrect threshold value;
- an off-by-one error.

Not every numeric value requires `-1`, `0`, and `+1` around it. Choose boundaries where the specification says behavior changes.

### 8.3 Invalid-input cases

Invalid-input tests verify that the unit rejects data outside its contract.

Example:

| Input | Expected behavior |
| --- | --- |
| `orderTotal = -1` | Throw `IllegalArgumentException` |

Other domains may require cases such as:

- `null` references;
- blank text;
- values outside an allowed range;
- unsupported enum values;
- inconsistent combinations of fields.

Do not invent validation rules that are absent from the requirements. A test should encode an intended contract, not a developer's guess.

### 8.4 Exceptional behavior

Some operations are specified to fail by throwing an exception. The exception is then part of the observable contract.

A useful test should verify the important part of that contract, for example:

- the exception type;
- possibly its message or data, if callers rely on them.

An exceptional path is not necessarily a system defect. Rejecting invalid input with the specified exception can be correct behavior.

### 8.5 Special business cases

Rules often contain exceptions or priority conditions.

In the calculator, a premium customer receives free delivery even when the order total is below the standard threshold. This case deserves its own test because it represents a separate business rule.

### 8.6 Regression cases

When a defect is found, add a test that reproduces it before or while fixing the code. The test should fail for the incorrect implementation and pass after the correction.

This converts knowledge about the defect into permanent automated protection.

### A compact test-case table

Before coding, a developer could prepare the following table:

| Scenario | Input | Expected behavior | Category |
| --- | --- | --- | --- |
| Ordinary small order | `2_000`, non-premium | Fee is `500` | Happy path |
| Just below threshold | `4_999`, non-premium | Fee is `500` | Boundary |
| Exactly at threshold | `5_000`, non-premium | Fee is `0` | Boundary |
| Just above threshold | `5_001`, non-premium | Fee is `0` | Boundary |
| Small premium order | `100`, premium | Fee is `0` | Special rule |
| Negative total | `-1`, either customer type | Throw `IllegalArgumentException` | Invalid input / exceptional behavior |

The table is not automatically a perfect test suite. It is a reasoned starting point based on the contract.

---

## 9. Avoiding redundant and misleading test cases

More tests do not automatically mean better protection.

### Redundant cases

If the implementation has the same behavior for every ordinary value from `0` to `4_999`, testing hundreds of arbitrary values may add little confidence compared with a representative ordinary value and relevant boundaries.

For example, these cases may be mostly redundant:

- `1_000` → `500`;
- `2_000` → `500`;
- `3_000` → `500`;
- `4_000` → `500`.

They all exercise the same rule. Their value depends on whether the domain contains additional distinctions between them.

### Tautological tests

A test is weak if it calculates the expected result by repeating the same algorithm as the production code.

Weak idea:

```java
int expected = premiumCustomer || orderTotal >= 5_000 ? 0 : 500;
int actual = calculator.calculate(orderTotal, premiumCustomer);
```

If both the test and production code contain the same misunderstanding, the test may pass while the behavior is wrong. Prefer simple, explicit expected values derived from the requirement.

### Tests without assertions

A method that calls production code but never checks a result proves very little. The absence of an unexpected exception is not enough unless “does not throw” is the behavior being tested.

Printing the result is also not an automated verification:

```java
System.out.println(calculator.calculate(5_000, false));
```

A human must inspect the output, so this is not a self-checking unit test.

---

## 10. Common misconceptions

### “A unit always means one method”

Not necessarily. A unit is a focused piece of behavior. A method is a common unit boundary, but the useful boundary depends on the design.

### “Every class needs a matching test class”

There is no requirement for a mechanical one-to-one mapping. Tests should be organized around meaningful behavior. A matching test class is a useful convention in many cases, not the definition of correctness.

### “Private methods must be tested directly”

Normally, test them through public behavior. If a private method contains substantial independent logic, consider extracting a separate component with a clear contract.

### “If all unit tests pass, the application works”

Passing unit tests show that the tested units behave as expected in the tested scenarios. They do not verify all integrations, configuration, infrastructure, or complete user workflows.

### “Code coverage proves test quality”

Coverage can reveal code that was not executed by tests. It cannot prove that assertions are correct, boundaries were selected well, or requirements were understood. Coverage is a diagnostic metric, not a substitute for reasoning.

### “Testing means testing only after implementation”

Tests can be written before, during, or after production code. In all cases, test-case design should begin with expected behavior rather than the current implementation.

---

## 11. Worked example: from requirement to test cases

Consider the following requirement:

> A password is acceptable when it contains at least 8 characters. A `null` password is invalid.

A possible implementation is:

```java
public final class PasswordValidator {

    public boolean isValid(String password) {
        return password != null && password.length() >= 8;
    }
}
```

### Step 1: Identify observable behavior

The public method returns `true` or `false`. Its internal condition is not the contract; the result for an input is the contract.

### Step 2: Find partitions and boundaries

Meaningful groups are:

- `null`;
- non-null strings shorter than 8 characters;
- strings with exactly 8 characters;
- strings longer than 8 characters.

The important length boundary is 8.

### Step 3: Select representative cases

| Scenario | Input | Expected result |
| --- | --- | --- |
| Missing password | `null` | `false` |
| Empty password | `""` | `false` |
| Just below minimum | `"1234567"` | `false` |
| Exactly minimum length | `"12345678"` | `true` |
| Above minimum length | `"123456789"` | `true` |

### Step 4: Check for invented assumptions

The requirement does not say that a password must contain uppercase letters, digits, or special characters. Tests should not require those rules unless the specification changes.

### Step 5: Check test value

The cases cover missing input, a special empty value, and both sides of the length boundary. Testing every possible eight-character string is unnecessary because the stated rule depends only on length.

---

## 12. Practice: classify the tests

For each example, decide whether it is primarily a unit, integration, or end-to-end test.

1. Call `TaxCalculator.calculate()` with a fixed amount and verify the returned tax.
2. Start the application with a test database and verify that `CustomerRepository` saves and loads a customer.
3. Open the application in a browser, sign in, place an order, and verify the confirmation page.
4. Call `EmailValidator.isValid()` with several strings and verify the returned boolean values.
5. Serialize an object using the application's configured JSON mapper and verify the produced JSON contract.

### Suggested answers

1. **Unit test** — it checks focused calculation logic directly.
2. **Integration test** — it intentionally verifies repository and database integration.
3. **End-to-end test** — it exercises a complete user workflow through the UI.
4. **Unit test** — it checks focused validation behavior.
5. Usually an **integration test** — it verifies application code together with actual serialization configuration. The exact label depends on the test boundary, but the integration is the subject.

---

## 13. Knowledge check

1. Why is a unit not always identical to one method?
2. What makes an automated unit test different from printing a result and inspecting it manually?
3. How do unit tests support refactoring?
4. Why is `5_000` an important test value for `DeliveryFeeCalculator`?
5. What does it mean to test observable behavior?
6. Why should private helper methods normally not be tested directly?
7. Can passing unit tests prove that a complete application works correctly? Why or why not?
8. What is the main idea of the testing pyramid?
9. Give one example each of a happy path, a boundary case, and invalid input.
10. Why can a large number of redundant tests make a test suite worse rather than better?

### Answers

1. A unit is a focused piece of behavior. One public behavior may use several methods, while one method may contain several scenarios that need separate tests.
2. An automated test compares actual and expected behavior and reports pass or failure without human inspection.
3. They check that covered observable behavior remains unchanged while the internal structure is modified.
4. It is the exact boundary where a non-premium customer's delivery fee changes from `500` to `0`.
5. It means verifying results or effects available through the unit's contract rather than checking internal implementation steps.
6. Direct private-method tests couple the test suite to implementation details and make refactoring more difficult. The behavior should normally be exercised through the public contract.
7. No. Unit tests do not by themselves verify integration, configuration, infrastructure, or complete workflows, and they cover only selected scenarios.
8. Use many fast, low-level tests and fewer broad, slower tests, while keeping a balanced suite of different test levels.
9. Examples will vary. For the delivery calculator: `2_000` is a happy path, `5_000` is a boundary case, and `-1` is invalid input.
10. Redundant tests increase maintenance and execution cost without necessarily adding meaningful confidence. They can also obscure the behaviors that matter.

---

## 14. Key takeaways

- A unit is a small, focused piece of observable software behavior.
- A unit test automatically checks that behavior in a controlled context.
- Unit tests provide fast feedback, reduce regression risk, support refactoring, and document behavior through examples.
- Unit tests form the broad base of the testing pyramid, but they do not replace integration or end-to-end tests.
- Good tests focus on public contracts and meaningful business behavior rather than private implementation details.
- Useful test suites cover normal behavior, boundaries, invalid input, exceptional behavior, and important business cases.
- Test-case selection requires reasoning. A high test count alone does not guarantee test quality.

---

## 15. JUnit version note

The code example uses the JUnit Jupiter API from the `org.junit.jupiter.*` packages. This programming model is used by both JUnit 5 and JUnit 6.

For this course:

- examples target **Java 25 and JUnit 6**;
- the relevant source code is also compatible with modern JUnit 5 releases unless a version-specific note says otherwise;
- JUnit 6 requires Java 17 or later, so it can be used with Java 25;
- setup, test anatomy, assertions, and execution will be covered in the next topic.

---

## Further reading

- [JUnit 6 User Guide: Overview](https://docs.junit.org/6.1.2/overview.html)
- [JUnit 6 User Guide: Writing Tests](https://docs.junit.org/6.1.2/writing-tests/intro.html)
- [Martin Fowler: Test Pyramid](https://martinfowler.com/bliki/TestPyramid.html)
- [Ham Vocke: The Practical Test Pyramid](https://martinfowler.com/articles/practical-test-pyramid.html)