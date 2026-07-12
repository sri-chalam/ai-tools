---
name: junit-guidelines
description: >
  JUnit test guidelines — invoke before planning, writing, changing, reviewing, or auditing JUnit tests (coverage, quality, or test-double usage) across a Java codebase, including auditing production classes that can't be swapped out for tests.
version: "0.1.0"
applyTo:
  - "src/test/**/*.java"
  - "**/*Test.java"
  - "**/*Tests.java"
tags:
  - junit
  - guidelines
maintainers:
  - name: Sri Chalam
category:
  - development
license: UNLICENSED
disable-model-invocation: false
---

You are a Java test engineer following these guidelines for all JUnit 5 test generation.

## Prerequisites
- JUnit 5 (Jupiter) is present in the application classpath
- `junit-jupiter-params` is required for parameterized tests (bundled with the `junit-jupiter` aggregate artifact)

**Assertion library:** Default to AssertJ (`assertThat`, `assertThatThrownBy`, `.as()`) when `assertj-core` is on the classpath. If the project uses only JUnit 5 assertions, substitute the JUnit equivalents — `assertEquals`/`assertTrue`/`assertThrows` — and replace `.as("msg")` with the message parameter available on each JUnit assertion method.

**Terminology:** *Test double* — any test replacement for a real dependency. *Stub* — a Mockito double with forced return values (`when/thenReturn`). *Fake* — a lightweight working implementation with in-memory state (Rule 8). *Mock* used loosely in this document means a Mockito double.

## Core Testing Philosophy

Write tests that are:
- **Fast, deterministic, isolated, and clear** — see Rule 2 (FIRST Principles) for the operational rules
- **Maintainable**: Only change when requirements change, not during refactoring
- **Treat tests as specifications of required behavior** - Tests document what the system must do, forming a contract that persists across refactorings.
- **Test observable behavior through public APIs, not implementation details** - When refactoring internal logic, well-written tests should remain stable because they verify outcomes rather than how those outcomes are achieved.
- **Write tests that validate a specific behavior or outcome, not just exercise a method.** Each test should represent one complete scenario with a clear expected result.
- **A test that cannot catch a real bug should not be written.** If a method contains no conditional logic, transformation, or error handling and only forwards its arguments to a dependency, skip the test — it verifies Mockito wiring, not application behavior.
- **Extract repeated test data to named constants**: Any identifier, code, or string used in more than one test method should be declared as a `public static final` constant with a UUID-like value. This gives test data a semantic name and a single point of change.
- **Never mock value objects, data classes, or pure in-process logic** — construct them for real. This includes Money-style value types, DTOs/records, dates and IDs, collections, mappers/converters, and validators. Mock only dependencies that cross a process boundary (see Rule 7).

---

## Rule 0: Plan Tests Before Writing Code

**Rationale:** Nothing governs *which* tests get written until a reviewer checks coverage after the fact. Without a plan, generation defaults to one happy-path test per method. Planning forces classification and enumeration to happen once, before any code exists.

**ALWAYS:**
- Classify each public method of the class under test as **logic-owning** (has its own conditional logic, transformation, or computation) or **orchestrating** (only wires calls to other logic-owning methods) before enumerating tests
- For each logic-owning method, list one Given/When/Then one-liner per branch, boundary, null/empty case, and error path
- For each orchestrating method, list only 1–2 representative wiring scenarios
- Order the plan: error paths and boundaries first, then happy paths — one happy-path test per behavior
- For every planned test, answer *"what specific production bug would make this test fail?"* Drop any test with no plausible answer
- Present the plan (test name → behavior) before generating code, then proceed to generate tests
- If a dependency has no interface seam and none is already planned, stub it with Mockito and move on — don't refactor production code to enable a fake unless that refactor is already in scope

**NEVER:**
- Add a test to the plan solely because "more coverage is better" — every entry must survive the bug-catching question above
- Generate multiple tests that assert the same behavior with different input literals — flag these for `@ParameterizedTest` consolidation in the plan, not as separate plan rows

**Example of a Test Plan**

| Test name | Behavior |
|---|---|
| `cardValidation_nullCardNumber_throwsIllegalArgumentException` | error path: null input |
| `cardValidation_invalidLuhnChecksum_returnsInvalid` | boundary: fails checksum |
| `paymentProcessing_gatewayTimeout_returnsDeclinedWithReason` | error path: gateway failure |
| `paymentProcessing_chargeValidCard_storesReceiptAndReturnsTransactionId` | happy path |

Each row states the behavior under test before any test code exists, so scope isn't decided
ad hoc mid-generation.

---

## Rule 1: General Test Guidelines
**ALWAYS:**
- Test edge cases and error conditions
- Test state transitions and business logic

**NEVER:**
- Test basic Java/library functionality (e.g., getters/setters, equals/hashCode unless custom logic)
- Test framework behavior (e.g., Spring's dependency injection)
- Test auto-generated code
- Test method call sequences unless absolutely necessary for the specific scenario. Only verify interactions when the "how" matters
- Test pure-delegation methods that only forward arguments to a dependency with no logic — see "A test that cannot catch a real bug should not be written"
  in Core Testing Philosophy.

**Example of a trivial delegation test to avoid**
```java
// Production code — pure delegation, no logic
class CardPaymentService {
    private final PaymentGateway gateway;

    public CardPaymentService(PaymentGateway gateway) {
        this.gateway = gateway;
    }

    public PaymentResult charge(CreditCard card, BigDecimal amount) {
        return gateway.charge(card, amount);
    }
}

// ❌ ANTI-PATTERN: A test that cannot catch a real bug should not be written — this proves only that Mockito returns what you told it to
class CardPaymentServiceTest {
    @Test
    void cardPayment_chargeValidCard_returnsDelegatedResult() {
        // Given
        PaymentGateway mockGateway = mock(PaymentGateway.class);
        CreditCard card = new CreditCard("4532015112830366", "12/25", "123");
        BigDecimal amount = new BigDecimal("99.99");
        PaymentResult expected = new PaymentResult(PaymentStatus.APPROVED, "AUTH123");
        when(mockGateway.charge(card, amount)).thenReturn(expected);
        CardPaymentService service = new CardPaymentService(mockGateway);

        // When
        PaymentResult result = service.charge(card, amount);

        // Then
        assertThat(result).isEqualTo(expected);  // tests the stub, not the application
    }
}
```

---

## Rule 2: Follow the FIRST Principles
**ALWAYS:**
- Execute in milliseconds to encourage frequent runs during development
- No dependencies on other tests or external factors like databases
- Produce consistent results regardless of environment or execution order
- Automatically verify pass/fail without manual inspection
- Write or update tests in the same change that introduces or modifies the behavior — never defer coverage to a later step

**Examples of FIRST Principles:**
***Examples of FAST principle:***
```java
// ✅ GOOD EXAMPLE: FAST: This test executes in milliseconds by mocking external payment gateway
// instead of making real network calls which would take seconds. 
@Test
void paymentAuthorization_authorizeValidCard_returnsApproved() {
    // Given - Use mocks to avoid slow external calls
    PaymentGateway mockGateway = mock(PaymentGateway.class);
    when(mockGateway.authorize(any(), any()))
        .thenReturn(new AuthResponse("AUTH123", AuthStatus.APPROVED));
    
    PaymentAuthorizationService service = new PaymentAuthorizationService(mockGateway);
    CreditCard card = new CreditCard("4532015112830366", "12/25", "123");
    Money amount = Money.dollars(250.00);

    // When
    AuthorizationResult result = service.authorize(card, amount);

    // Then
    assertThat(result.getStatus()).isEqualTo(AuthStatus.APPROVED);
}

```

***INDEPENDENT principle:*** Each test sets up its own data and passes in any order — see Rule 10 (Keep Tests Independent) for examples.

***Examples of REPEATABLE principle:***
```java
// ✅ GOOD EXAMPLE: REPEATABLE: Test produces same results every time, regardless of environment.
// No dependency on current date, random values, or external systems.
class CurrencyConverterTest {
    @Test
    void feeCalculation_calculateForeignExchangeFee_returnsFixedPercentageFee() {
        // Given
        FeeCalculator calculator = new FeeCalculator();
        Money amount = Money.dollars(1000.00);

        // When
        BigDecimal fee = calculator.calculateForeignExchangeFee(amount);

        // Then - deterministic input yields a fixed expected value, every run
        assertThat(fee).isEqualTo(new BigDecimal("30.00")); // 3% fee
    }
}

// ❌ BAD EXAMPLE: Non-repeatable test (ANTI-PATTERN)
class BadCurrencyConverterTest {   
    @Test
    void currencyConversion_convertWithLiveExchangeRate_returnsNonDeterministicResult() {
        // Given
        LocalDate today = LocalDate.now();        // BAD: changes daily
        ExchangeRateService realService = new LiveExchangeRateService(); // BAD: real HTTP call
        Random random = new Random();            // BAD: different each run
        BigDecimal randomAmount = new BigDecimal(random.nextDouble() * 1000);
        CurrencyConverter converter = new CurrencyConverter(realService);
        
        // When
        Money result = converter.convert(
            new Money(randomAmount, Currency.USD), 
            Currency.EUR
        );
        
        // Then - assertion might pass today but fail tomorrow
        // assertTrue(result.getAmount().compareTo(new BigDecimal("800")) > 0);
    }
}

```

***Examples of SELF-VALIDATING principle:***
```java
// ✅ GOOD EXAMPLE: SELF-VALIDATING: Test automatically determines pass/fail without manual inspection.
// No need to check logs, databases, or console output.
class FraudDetectionServiceTest {
    @Test
    void fraudDetection_analyzeHighRiskTransaction_flagsAsHighRisk() {
        // Given
        FraudDetectionService fraudService = new FraudDetectionService();
        
        Transaction suspiciousTransaction = Transaction.builder()
            .amount(new BigDecimal("9999.99"))
            .cardNumber("4532015112830366")
            .merchantCountry("NG") // High-risk country
            .transactionTime(LocalTime.of(3, 30)) // Unusual hour
            .isOnlineTransaction(true)
            .customerLocation("US")
            .build();
        
        // When
        FraudScore score = fraudService.analyze(suspiciousTransaction);
        
        // Then - Clear pass/fail without manual checking
        assertThat(score.isHighRisk()).as("Transaction should be flagged as high risk").isTrue();
        assertThat(score.getScore()).as("Fraud score should exceed 75").isGreaterThan(75);
        assertThat(score.getRiskFactors())
            .contains("HIGH_AMOUNT", "HIGH_RISK_COUNTRY", "UNUSUAL_TIME");
    }
}

```

- **Timely:** Write or update tests in the same change that introduces or modifies the behavior — never defer coverage to a later step.

---

## Rule 3: Avoid Testing Implementation Details
**Rationale:** Tests using public APIs are resilient to refactoring and won't break when internal implementation changes.

**ALWAYS:**
- Test the "what" not the "how"
- Access the system under test the same way real users would
- Test the complete behavior through the public interface
- Tests that break during refactoring indicate they weren't written at the appropriate abstraction level

**NEVER:**
- Test private methods directly
- Use reflection to access private members for testing
- Make private methods package-private just for testing

**Examples of Avoiding Testing Implementation Details**
```java
// ❌ BAD EXAMPLE: Testing the internal Luhn checksum calculation method (ANTI-PATTERN)
@Test
void cardValidation_calculateLuhnChecksumDirectly_returnsZeroForValidCard() {
    // Given
    CreditCardValidator validator = new CreditCardValidator();
    String cardNumber = "4532015112830366";

    // When - calculateLuhnChecksum() is a private helper method; tests HOW, not WHAT
    int checksum = validator.calculateLuhnChecksum(cardNumber);

    // Then
    assertThat(checksum).isEqualTo(0);
}

// ✅ GOOD EXAMPLE: Testing public behavior with invalid card
@Test
void cardValidation_validateCardWithInvalidLuhnChecksum_returnsInvalid() {
    // Given
    CreditCardValidator validator = new CreditCardValidator();
    String invalidCard = "4532015112830367"; // Last digit wrong

    // When
    boolean result = validator.isValid(invalidCard);

    // Then - Don't care HOW it validates, just that it rejects invalid cards
    assertThat(result).isFalse();
}

// ✅ GOOD EXAMPLE: Testing public API — same behavior across card networks,
// consolidated into one parameterized test instead of three copies differing
// only in input literals
@ParameterizedTest(name = "{1} card number {0} is valid")
@CsvSource({
    "4532015112830366, Visa",
    "5555555555554444, Mastercard",
    "378282246310005,  Amex"
})
void cardValidation_validateCardNumberOfEachNetwork_returnsValid(String cardNumber, String network) {
    // Given
    CreditCardValidator validator = new CreditCardValidator();

    // When
    boolean result = validator.isValid(cardNumber);

    // Then
    assertThat(result).as("%s card should be valid", network).isTrue();
}

// ✅ GOOD EXAMPLE: Testing validation result, not implementation
@Test
void cardValidation_validateCardWithInvalidFormat_returnsValidationErrors() {
    // Given
    CreditCardValidator validator = new CreditCardValidator();
    String invalidCard = "1234567890123456";

    // When
    ValidationResult result = validator.validate(invalidCard);

    // Then - Don't test HOW it determined invalidity, just THAT it did
    assertThat(result.isValid()).isFalse();
    assertThat(result.getErrors()).contains("Invalid card number format");
}
```

---

## Rule 4: Use Descriptive Test Names - Behavior, Action, Expected result
**Rationale:**
Test names are often the first thing visible in failure reports. Clear names communicate both the action and expected outcome, making debugging faster.

**ALWAYS:**
- Follow the exact three-segment, two-underscore pattern: `methodUnderTest_context_expectedResult`
- **`methodUnderTest`** — the method being exercised
- **`context`** — the single most distinguishing condition in camelCase; let the `// Given` block carry secondary details
- **`expectedResult`** — the observable outcome (`returnsTrue`, `returnsFalse`, `throwsIllegalArgumentException`, etc.)
- Use descriptive names even if verbose

**NEVER:**
- Use more than two underscores — if context feels long, trim to its key phrase, do not split into a fourth segment

**Examples of Descriptive Test Names**
```java
class CreditCardValidatorTest {
    // ❌ BAD EXAMPLE: Only mentions the action, not the expected outcome (ANTI-PATTERN)
    @Test
    void testValidation() {
        // Given
        CreditCardValidator validator = new CreditCardValidator();

        // When
        boolean result = validator.isValid("1234");

        // Then
        assertThat(result).isFalse();
    }

    // ✅ GOOD EXAMPLE: Complete behavior description with edge case
    @Test
    void cardValidation_validateCorrectLengthWithInvalidLuhn_returnsInvalid() {
        // Given
        CreditCardValidator validator = new CreditCardValidator();
        String invalidChecksum = "4532015112830367"; // Wrong last digit

        // When
        boolean result = validator.isValid(invalidChecksum);

        // Then
        assertThat(result).isFalse();
    }
}
```

---

## Rule 5: Avoid Logic in Tests
**Rationale:** Tests should contain minimal logic; complex test logic indicates the test or production code needs refactoring.
**ALWAYS:**
- Keep test bodies as simple, linear statements — no conditionals (`if/else`), loops (`for, while`), or complex expressions

**Examples of Avoiding Logic in Tests**
```java
// ✅ GOOD EXAMPLE - Simple, clear test
@Test
void refundProcessing_refundValidTransaction_returnsCompleted() {
    // Given
    RefundProcessor processor = new RefundProcessor();
    Transaction originalTransaction = createTransaction("99.99");

    // When
    RefundResult result = processor.refund(originalTransaction);

    // Then
    assertThat(result.getStatus()).isEqualTo(RefundStatus.COMPLETED);
}

// ❌ BAD EXAMPLE: AVOID - Logic in test - (ANTI-PATTERN)
@Test
void refundProcessing_processRefundWithConditionalLogic_obscuresIntent() {
    // Given
    RefundProcessor processor = new RefundProcessor();

    // When - BAD: Complex loops and conditionals make tests hard to understand
    for (int i = 0; i < 10; i++) {
        if (i % 2 == 0) {
            // This is a code smell
        }
    }

    // Then - no clear assertion
}
```

## Rule 6: Use Setup Methods Appropriately (@BeforeEach and @BeforeAll)
**Rationale:** Leverage JUnit's setup annotations to reduce test duplication while maintaining test independence and clarity.
Setup methods help initialize common test dependencies and data, but must be used carefully to avoid hidden dependencies and maintain test readability.

**ALWAYS:** 
- Use @BeforeEach for per-test setup that ensures each test starts with a fresh state
- Use @BeforeAll for expensive one-time setup of immutable shared resources
- Keep setup methods focused and minimal
- Document non-obvious setup behavior

**Examples of @BeforeEach**
```java
class PaymentProcessorTest {
    private PaymentProcessor processor;
    private PaymentGateway mockGateway;
    private CreditCard validCard;
    
    // ✅ GOOD EXAMPLE: @BeforeEach creates fresh instances for each test
    @BeforeEach
    void setUp() {
        // Each test gets its own fresh instances
        mockGateway = mock(PaymentGateway.class);
        processor = new PaymentProcessor(mockGateway);
        
        // Fresh card object for each test
        validCard = new CreditCard(
            "4532015112830366",
            "12/26",
            "123"
        );
    }
    
    @Test
    void paymentProcessing_processValidCard_returnsSuccess() {
        // Given
        when(mockGateway.authorize(any())).thenReturn(
            new GatewayResponse("APPROVED", "AUTH123")
        );

        // When
        PaymentResult result = processor.processPayment(
            validCard,
            new BigDecimal("99.99")
        );

        // Then
        assertThat(result.getStatus()).isEqualTo(PaymentStatus.SUCCESS);
    }

    @Test
    void paymentProcessing_processWithGatewayTimeout_retriesAndSucceeds() {
        // Given
        when(mockGateway.authorize(any()))
            .thenThrow(new GatewayTimeoutException())
            .thenReturn(new GatewayResponse("APPROVED", "AUTH456"));

        // When
        PaymentResult result = processor.processPayment(
            validCard,
            new BigDecimal("50.00")
        );

        // Then
        assertThat(result.getStatus()).isEqualTo(PaymentStatus.SUCCESS);
        verify(mockGateway, times(2)).authorize(any());
    }    
}
```

**Examples of @BeforeAll**
```java
class CreditCardValidatorTest {
    private static CardNetworkRules networkRules;
    private static BINDatabase binDatabase;
    private CreditCardValidator validator;
    
    // ✅ GOOD EXAMPLE: @BeforeAll for expensive one-time setup of immutable data
    @BeforeAll
    static void setUpOnce() {
        // Load card network rules once (expensive operation)
        networkRules = CardNetworkRules.loadFromFile("card-network-rules.json");
        
        // Initialize BIN database once (large dataset)
        binDatabase = BINDatabase.loadFromCSV("bin-ranges.csv");
        
        // These are immutable and can be safely shared across all tests
    }
    
    @BeforeEach
    void setUp() {
        // Create fresh validator for each test, using shared immutable rules
        validator = new CreditCardValidator(networkRules, binDatabase);
    }
    
    @Test
    void cardValidation_validateVisaCard_returnsValid() {
        // Given
        String visaCard = "4532015112830366";

        // When
        boolean result = validator.isValid(visaCard);

        // Then
        assertThat(result).isTrue();
    }

    @Test
    void binLookup_identifyCardIssuer_returnsCorrectBank() {
        // Given
        String cardNumber = "4532015112830366";

        // When
        CardIssuer issuer = validator.identifyIssuer(cardNumber);

        // Then
        assertThat(issuer.getBankName()).isEqualTo("Chase Bank");
        assertThat(issuer.getNetwork()).isEqualTo(CardNetwork.VISA);
    }
}
```

---

## Rule 7: Mock External Dependencies
**Rationale:** Isolate the unit under test from external systems like AWS (Cloud) Services, databases, APIs, and file systems — with a mocking framework, or with a fake per Rule 8 when an interface seam exists. This keeps tests fast, reliable, and prevents test failures due to external system issues.

In this rule, "mock" means a Mockito double used to **stub return values** (`when/thenReturn`) so the test can assert the SUT's observable outcome. Use `verify()` only when the side effect *is* the contract and leaves no observable state or return value to assert (e.g. `sendEmail`, `publishEvent`) — never verify calls whose result you can already assert.

**ALWAYS:**
- Stub external services only (mock the dependency, stub its methods with `when/thenReturn`)
  - Message queues/brokers (Kafka, SQS, SNS, etc.)
  - Cache systems (Redis, Memcached, ElastiCache)
  - Third-party libraries that make network calls (payment gateways, email services, etc.)
  - Databases (DynamoDB, Postgres, MySQL, etc.)
  - Cloud storage services (S3)
  - File systems


**Where to place the double when a dependency itself calls an external system:**
When the class under test (A) depends on an application class (B) that in turn makes an external call (REST API, DB, queue):
- **If B is a thin client/gateway** — its only job is making the external call and mapping the response — B *is* the boundary: stub B in A's tests (or fake it per Rule 8 if it's stateful or stubbed identically across many tests)
- **If B contains its own business logic**, use the **real B** in A's tests and place the double one level deeper, at B's own client/gateway seam — stubbing B directly hard-codes assumptions about B's behavior that nothing verifies
- **Never** double below the application boundary — do not mock third-party client types such as `RestTemplate`, `WebClient`, `HttpClient`, `S3Client`
- **Fallback:** if B is logic-owning but exposes no injectable seam and refactoring is out of scope, stub B directly and note the design smell (logic mixed with I/O)

**Examples of Mocking External Dependencies**
```java
// ✅ GOOD EXAMPLE: Mocking external payment gateway API
@Test
void paymentProcessing_chargeValidCard_returnsSuccessfulResult() {
    // Given
    PaymentGateway mockGateway = mock(PaymentGateway.class);
    CreditCard card = new CreditCard("4532015112830366", "12/25", "123");
    when(mockGateway.charge(eq(card), eq(new BigDecimal("99.99"))))
        .thenReturn(new GatewayResponse("SUCCESS"));
    PaymentService service = new PaymentService(mockGateway);

    // When
    PaymentResult result = service.processPayment(card, new BigDecimal("99.99"));

    // Then - assert the observable outcome, not the interaction
    assertThat(result.getStatus()).isEqualTo(PaymentStatus.SUCCESS);
}


// ❌ BAD EXAMPLE: Making real external calls (ANTI-PATTERN)
@Test
@Disabled("This test makes real external calls - DO NOT DO THIS")
void paymentProcessing_processPaymentWithRealAwsServices_causesNetworkDependency() {
    // Given
    DynamoDbClient realDynamoDb = DynamoDbClient.builder() // BAD: real DynamoDB client
        .region(Region.US_EAST_1)
        .build();
    S3Client realS3 = S3Client.builder()                   // BAD: real S3 client
        .region(Region.US_EAST_1)
        .build();
    PaymentProcessor processor = new PaymentProcessor(realDynamoDb, realS3);
    CreditCard card = new CreditCard("4532015112830366", "12/26", "123");
    PaymentRequest request = new PaymentRequest(card, new BigDecimal("99.99"));

    // When - BAD: real network calls to AWS — slow, flaky (depends on AWS
    // availability and credentials), and pollutes real data stores
    processor.processPayment(request);

    // Then - no assertions; result depends on external state
}
```

---

## Rule 8: Use Interface-Based Fake Implementations for Stateful Complex External Dependencies
**Rationale:** For complex, stateful external service dependencies, prefer fake implementations over mocking frameworks. Fakes provide realistic behavior, are reusable across tests, and result in more maintainable test suites compared to mocks. When an external dependency is indirectly used (not directly injected), fake implementations provide a simpler and more maintainable testing approach.

**Resolving overlap with Rule 7:** A stateful dependency (DB, queue, S3) can look like it qualifies for both rules. Prefer this rule's fake approach only when an interface seam already exists or is trivial to add. "Trivial to add" means ALL of the following hold:
- The dependency is already injected via constructor (no `new` inside the class under test)
- The class under test uses only 1–3 methods of the dependency
- Extracting the interface requires changing only the dependency's class declaration and the injection point — no call sites elsewhere in production code change

Otherwise mock per Rule 7 rather than introducing a new seam just to enable a fake.

**Design-smell heuristic (for audits):** flag any class that directly depends on a third-party client type (`S3Client`, `DynamoDbClient`, `RestTemplate`, `WebClient`, `RestClient`, etc.) with no application-owned interface between it and its callers — it can only ever be mocked, never faked, without a refactor first.

**ALWAYS:**
- Prefer interface-based design for testability
- When refactoring is feasible, prefer fakes over mocks for better maintainability
- Use dependency injection to allow swapping real implementations with fakes during testing
- Fakes centralize implementation in one place; mocks scatter configuration across multiple test files
- Fall back to a mocking framework when an interface seam isn't feasible — Mockito for standard scenarios, PowerMock only as a last resort for static, final, or private members Mockito can't handle

**Examples of Interface-Based Fake Implementations**
```java
// Interface for S3 storage operations
interface S3StorageService {
    boolean storeReceipt(String transactionId, String receiptContent);
}

// Production code using interface (enables testing with fakes)
class CardPaymentProcessor {
    private final S3StorageService s3Service;

    public CardPaymentProcessor(S3StorageService s3Service) {
        this.s3Service = s3Service;
    }

    public String processPayment(String cardNumber, double amount) {
        if (!isValidCard(cardNumber) || amount <= 0) {
            return null;
        }
        String transactionId = "TXN-" + System.currentTimeMillis();
        String receipt = generateReceipt(transactionId, cardNumber, amount);
        boolean stored = s3Service.storeReceipt(transactionId, receipt);
        return stored ? transactionId : null;
    }

    private boolean isValidCard(String cardNumber) {
        return cardNumber != null && cardNumber.length() >= 13;
    }

    private String generateReceipt(String transactionId, String cardNumber, double amount) {
        String maskedCard = "****-" + cardNumber.substring(cardNumber.length() - 4);
        return String.format("Transaction: %s\nCard: %s\nAmount: $%.2f",
                           transactionId, maskedCard, amount);
    }
}

// ✅ GOOD EXAMPLE: Fake implementation with in-memory state
class FakeS3StorageService implements S3StorageService {
    private final Map<String, String> storage = new HashMap<>();

    @Override
    public boolean storeReceipt(String transactionId, String receiptContent) {
        if (transactionId == null || receiptContent == null) {
            return false;
        }
        storage.put(transactionId, receiptContent);
        return true;
    }

    public String getReceipt(String transactionId) {
        return storage.get(transactionId);
    }
}

// ✅ GOOD EXAMPLE: Test using fake implementation
@Test
void paymentProcessing_processValidCardPayment_storesReceiptAndReturnsTransactionId() {
    // Given
    FakeS3StorageService fakeS3 = new FakeS3StorageService();
    CardPaymentProcessor processor = new CardPaymentProcessor(fakeS3);

    // When
    String transactionId = processor.processPayment("4532123456789010", 99.99);

    // Then
    assertThat(transactionId).isNotNull();
    String receipt = fakeS3.getReceipt(transactionId);
    assertThat(receipt).contains(transactionId);
    assertThat(receipt).contains("$99.99");
}

// ❌ BAD EXAMPLE: Tightly coupled to AWS SDK (ANTI-PATTERN)
class BadCardPaymentProcessor {
    public String processPayment(String cardNumber, double amount) {
        // BAD: Direct dependency on AWS S3 client - hard to test
        S3Client s3Client = S3Client.builder()
            .region(Region.US_EAST_1)
            .build();

        String transactionId = "TXN-" + System.currentTimeMillis();
        String receipt = generateReceipt(transactionId, cardNumber, amount);

        // BAD: Direct S3 call - requires mocking framework or real AWS access
        s3Client.putObject(PutObjectRequest.builder()
            .bucket("receipts")
            .key(transactionId)
            .build(),
            RequestBody.fromString(receipt));

        return transactionId;
    }
}
```

---

## Rule 9: Test for Expected Exceptions
**Rationale:** Verify that code throws appropriate exceptions for invalid inputs or error conditions. This ensures proper error handling and validates that your code fails gracefully with meaningful error messages.

**ALWAYS:**
- Use assertThatThrownBy for exception testing
- Verify exception type and message
- Test both happy path and error scenarios

**NEVER:**
- Use try-catch blocks in tests instead of assertThatThrownBy

**Examples of Testing Expected Exceptions**
```java
// ✅ GOOD EXAMPLE: Validate exception type and message
@Test
void cardValidation_validateEmptyCardNumber_throwsIllegalArgumentException() {
    // Given
    CreditCardValidator validator = new CreditCardValidator();

    // When / Then
    assertThatThrownBy(() -> validator.isValid(""))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessage("Card number cannot be empty");
}

```
---

## Rule 10: Keep Tests Independent
**Rationale:** Tests must not depend on execution order or the results of other tests to ensure reliability. Each test should be completely self-contained and produce consistent results regardless of when or in what order it runs.

**ALWAYS:**
- Each test should set up its own data
- Use @BeforeEach for common setup to ensure fresh state
- Avoid shared mutable state between tests
- Tests should pass in any order

**NEVER:**
- Leave side effects that affect other tests

**Examples of Keeping Tests Independent**
```java
// ✅ GOOD EXAMPLE: Independent tests with fresh setup
class TransactionProcessorTest {
    private TransactionProcessor processor;
    private CreditCard testCard;

    @BeforeEach
    void setUp() {
        // Each test gets fresh instances
        processor = new TransactionProcessor();
        testCard = new CreditCard("4532015112830366", "12/25", "123");
    }

    @Test
    void transactionProcessing_authorizeValidAmount_returnsAuthorizedAmount() {
        // Given - processor and testCard initialized in @BeforeEach

        // When
        BigDecimal result = processor.authorize(testCard, new BigDecimal("50.00"));

        // Then
        assertThat(result).isEqualTo(new BigDecimal("50.00"));
    }

    @Test
    void transactionProcessing_authorizeDifferentAmount_returnsAuthorizedAmount() {
        // Given - processor and testCard initialized in @BeforeEach

        // When
        BigDecimal result = processor.authorize(testCard, new BigDecimal("100.00"));

        // Then
        assertThat(result).isEqualTo(new BigDecimal("100.00"));
    }
}

// ❌ BAD EXAMPLE: Tests sharing mutable state (ANTI-PATTERN)
class BadTransactionProcessorTest {
    // BAD: Shared mutable state across tests
    private static TransactionProcessor sharedProcessor = new TransactionProcessor();
    private static BigDecimal totalAmount = BigDecimal.ZERO;

    @Test
    @Order(1) // BAD: Test order matters - red flag!
    void transactionProcessing_accumulateFirstPayment_updatesSharedTotal() {
        // Given - no local setup; relies on class-level shared mutable state (BAD)

        // When
        totalAmount = totalAmount.add(new BigDecimal("50.00"));

        // Then
        assertThat(totalAmount).isEqualTo(new BigDecimal("50.00"));
    }

    @Test
    @Order(2) // BAD: Depends on first test running first
    void transactionProcessing_accumulateSecondPayment_dependsOnPreviousTestState() {
        // Given - BAD: assumes totalAmount is already $50.00 from the previous test

        // When
        totalAmount = totalAmount.add(new BigDecimal("100.00"));

        // Then - PROBLEM: Fails if first test doesn't run first
        assertThat(totalAmount).isEqualTo(new BigDecimal("150.00"));
    }
}
```
---

## Rule 11: Organize Tests Using Given-When-Then Structure
**Rationale:** Organize every test into three clear sections: Given (setup), When (action), Then (verification). This makes tests self-documenting and easy to understand at a glance. The Given-When-Then pattern provides a consistent structure that improves test readability and maintainability.

**ALWAYS:**
- Organize tests into three clear sections to enhance readability
  - **Given**: Establish preconditions and initial state
  - **When**: Execute the behavior being tested
  - **Then**: Verify expected outcomes
- Place Given setup in test methods rather than hiding in @BeforeEach when behavior-specific
- Use multiple When-Then pairs for multi-step validations when testing sequential operations
- Keep each section clearly separated with comments or blank lines

**Examples of Given-When-Then Structure**
```java
// ✅ GOOD EXAMPLE: Clear Given-When-Then structure for single behavior
@Test
void cardProcessing_processTransactionWithSufficientBalance_approvesTransaction() {
    // Given - Setup preconditions and initial state
    CreditCard card = new CreditCard("4532-1111-2222-3333", LocalDate.of(2027, 12, 31));
    card.setAvailableCredit(new BigDecimal("5000.00"));
    CardProcessor processor = new CardProcessor();
    BigDecimal purchaseAmount = new BigDecimal("100.00");

    // When - Execute the behavior being tested
    TransactionResult result = processor.processTransaction(card, purchaseAmount);

    // Then - Verify expected outcomes
    assertThat(result.isApproved()).isTrue();
    assertThat(result.getStatus()).isEqualTo("APPROVED");
    assertThat(card.getAvailableCredit()).isEqualTo(new BigDecimal("4900.00"));
}


// ❌ BAD EXAMPLE: No clear Given-When-Then structure, testing multiple behaviors (ANTI-PATTERN)
@Test
void cardProcessing_processMultipleTransactionScenarios_combinesUnrelatedBehaviors() {
    // Given
    CreditCard card = new CreditCard("4532-1111-2222-3333", LocalDate.of(2027, 12, 31));
    card.setAvailableCredit(new BigDecimal("5000.00"));
    CardProcessor processor = new CardProcessor();

    // When - BAD: Behavior 1 (sufficient balance) — each behavior belongs in its own test
    TransactionResult result1 = processor.processTransaction(card, new BigDecimal("100.00"));
    // Then
    assertThat(result1.isApproved()).isTrue();

    // When - BAD: Behavior 2 (insufficient funds) — unrelated to behavior 1
    TransactionResult result2 = processor.processTransaction(card, new BigDecimal("10000.00"));
    // Then
    assertThat(result2.isApproved()).isFalse();

    // When - BAD: Behavior 3 (expired card) — unrelated to behaviors 1 and 2
    card.setExpirationDate(LocalDate.of(2023, 1, 1));
    TransactionResult result3 = processor.processTransaction(card, new BigDecimal("50.00"));
    // Then
    assertThat(result3.isApproved()).isFalse();
}

// ❌ BAD EXAMPLE: Setup and action mixed together (ANTI-PATTERN)
@Test
void cardProcessing_processTransactionWithInlineSetup_mixesGivenAndWhen() {
    // Given / When - BAD: object construction and action collapsed into one expression;
    // no clear boundary between setup and the behavior being tested
    CardProcessor processor = new CardProcessor();
    TransactionResult result = processor.processTransaction(
        new CreditCard("4532-1111-2222-3333", LocalDate.of(2027, 12, 31))
            .setAvailableCredit(new BigDecimal("5000.00")),
        new BigDecimal("100.00")
    );

    // Then
    assertThat(result.isApproved()).isTrue();
}
```

---

## Rule 12: Write Descriptive Failure Messages
**Rationale:** When a test fails, the error message should immediately indicate what was expected versus what actually happened, without needing to debug. Clear failure messages save significant debugging time by providing context about the failure directly in the test output.

**ALWAYS:**
- Use assertion messages that describe what went wrong in business terms
- Include expected vs actual values with meaningful context
- Use AssertJ's `.as()` method or JUnit's message parameter to add descriptions
- Include relevant object state in failure messages when helpful
- Make failure messages readable by non-developers when possible

**NEVER:**
- Omit important context that would help diagnose the failure

**Examples of Writing Descriptive Failure Messages**
```java
// ❌ BAD EXAMPLE: Unclear failure message - only shows true/false (ANTI-PATTERN)
@Test
void processPayment_insufficientFunds_unclearMessage() {
    // Given
    DebitCard card = new DebitCard("6011123456789012", new BigDecimal("25.00"));
    PaymentProcessor processor = new PaymentProcessor();

    // When
    PaymentResult result = processor.processPayment(card, new BigDecimal("100.00"));

    // Then
    // BAD: Failure message would be: "expected: false but was: true"
    assertThat(result.isApproved()).isFalse();
}

// ✅ GOOD EXAMPLE: Clear failure message showing expected vs actual
@Test
void processPayment_insufficientFunds_clearMessage() {
    // Given
    DebitCard card = new DebitCard("6011123456789012", new BigDecimal("25.00"));
    PaymentProcessor processor = new PaymentProcessor();

    // When
    PaymentResult result = processor.processPayment(card, new BigDecimal("100.00"));

    // Then
    // GOOD: Failure message: "Expected payment to be DECLINED, but got status APPROVED with reason 'INSUFFICIENT_FUNDS'"
    assertThat(result.getStatus())
        .as("Expected payment to be DECLINED, but got status %s with reason '%s'",
            result.getStatus(), result.getDeclineReason())
        .isEqualTo(PaymentStatus.DECLINED);

    assertThat(result.getDeclineReason())
        .as("Insufficient funds for transaction")
        .isEqualTo("INSUFFICIENT_FUNDS");
}
```

---

## Rule 13: Test logic-owning methods exhaustively, orchestrating methods for wiring

**Rationale:** A method with its own conditional logic, format handling, or non-trivial computation must
be tested directly and exhaustively — one test per branch, per accepted input format, per significant
edge case. A method that orchestrates calls to such methods needs only a few representative scenarios
confirming the wiring: correct input is passed and the result is stored or returned correctly.
Testing wiring exhaustively is impractical; testing logic only through a caller leaves branches
silently uncovered.

**ALWAYS:**
- Test logic-owning methods directly with one test per branch and per accepted input format
- Test orchestrating methods with two or three representative scenarios — one per accepted input format is enough
- When a caller-level test fails, it should point to a wiring problem, not an edge case already proven at the logic level

**NEVER:**
- Rely on caller-level tests to cover all branches of a logic-owning method
- Duplicate exhaustive edge-case coverage at every caller level

**Examples**

```java
// ✅ GOOD: Logic-owning method tested directly for every branch and format.
// CardExpiryParser accepts "MM/YY" and "MM/YYYY" — each format, null, and
// invalid input each get their own test.
class CardExpiryParserTest {

    @Test
    void expiryParsing_parseShortYearFormat_returnsYearMonthDate() { ... }     // "12/25"

    @Test
    void expiryParsing_parseLongYearFormat_returnsYearMonthDate() { ... }      // "12/2025"

    @Test
    void expiryParsing_parseNullExpiry_returnsNull() { ... }

    @Test
    void expiryParsing_parseInvalidFormat_returnsNull() { ... }                // "25-12"

    @Test
    void expiryParsing_parseExpiredDate_returnsExpiredYearMonth() { ... }
}

// ✅ GOOD: Orchestrating method tested with representative scenarios only —
// edge cases are already proven in CardExpiryParserTest.
// Null expiry, invalid format, and expired card are tested here because
// they are business scenarios CardValidator must handle correctly,
// not because we are re-proving the parser's format logic.
class CardValidatorTest {

    @Test
    void cardValidation_validateCardWithShortYearExpiry_passesValidation() { ... }

    @Test
    void cardValidation_validateCardWithLongYearExpiry_passesValidation() { ... }

    @Test
    void cardValidation_validateCardWithNullExpiry_failsValidation() { ... }

    @Test
    void cardValidation_validateCardWithInvalidExpiry_failsValidation() { ... }

    @Test
    void cardValidation_validateCardWithExpiredDate_failsValidation() { ... }
}

// ❌ BAD: Re-testing all parser format variants through CardValidator —
// the "MM/YY" vs "MM/YYYY" branching logic is already proven in CardExpiryParserTest.
// These tests add no wiring proof; they only re-exercise the parser.
class BadCardValidatorTest {

    @Test
    void cardValidation_validateCardWithShortYearExpiry_passesValidation() { ... }   // duplicate

    @Test
    void cardValidation_validateCardWithLongYearExpiry_passesValidation() { ... }    // duplicate

    @Test
    void cardValidation_validateCardWithShortYearExpiryAndVisaCard_passesValidation() { ... }  // duplicate

    @Test
    void cardValidation_validateCardWithLongYearExpiryAndVisaCard_passesValidation() { ... }   // duplicate
}
```

---

## Final Step: Validate, Incorporate, Compile

### Step 1 — Delegate validation to the `junit-validator` subagent

**This step requires subagent support (Claude Code / Claude Agent SDK).** If you are running in an environment that cannot invoke subagents (e.g. an AI coding agent without this capability), **skip Step 1 and Step 2 entirely** and proceed to Step 3.

If subagents are supported, after writing all test methods invoke the `junit-validator` subagent to validate them in a fresh context. Pass:
- `testFiles` — absolute path(s) of the test file(s) you just wrote
- `classesUnderTest` — absolute path(s) of the class(es) under test
- `guidelinesPath` — the absolute path of this SKILL.md (so the reviewer reads the same rules)

> Requires the `junit-validator` subagent resolvable from `~/.claude/agents/
> It is read-only and returns a findings table — it does not edit code.

### Step 2 — Incorporate the findings

The subagent returns a findings table tagged **must-fix / should-fix / nit**.
- Apply every **must-fix** and **should-fix** finding yourself.
- Report **nit** findings to the user without changing code.

### Step 3 — Verify compilation (always runs)

After writing all test methods (and after incorporating any findings):
- Compile the test source to confirm there are no errors.
- Fix any compilation errors before completing — do not leave broken code.
- Repeat until the build is clean.
