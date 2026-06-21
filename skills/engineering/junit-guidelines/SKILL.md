---
name: junit-guidelines
description: >
  TRIGGER — invoke before writing, modifying, fixing, or reviewing any
  Java test. Covers FIRST principles, GWT structure, naming conventions,
  mocking strategy, and exception testing. Auto-applies to test source
  files via applyTo patterns.
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
---

You are a Java test engineer following these guidelines for all JUnit 5 test generation.

## Prerequisites
- JUnit 5 (Jupiter) is present in the application classpath

**Assertion library:** Default to AssertJ (`assertThat`, `assertThatThrownBy`, `.as()`) when `assertj-core` is on the classpath. If the project uses only JUnit 5 assertions, substitute the JUnit equivalents — `assertEquals`/`assertTrue`/`assertThrows` — and replace `.as("msg")` with the message parameter available on each JUnit assertion method.

## Core Testing Philosophy

Write tests that are:
- **Fast**: Execute in milliseconds, not seconds
- **Deterministic**: Always produce the same result
- **Isolated**: Test one behavior at a time
- **Maintainable**: Only change when requirements change, not during refactoring
- **Clear**: Easy to understand and debug
- **Treat tests as specifications of required behavior** - Tests document what the system must do, forming a contract that persists across refactorings.
- **Test observable behavior through public APIs, not implementation details** - When refactoring internal logic, well-written tests should remain stable because they verify outcomes rather than how those outcomes are achieved.
- **Write tests that validate a specific behavior or outcome, not just exercise a method.** Each test should represent one complete scenario with a clear expected result.
- **A test that cannot catch a real bug should not be written.** If a method contains no conditional logic, transformation, or error handling and only
  forwards its arguments to a dependency, skip the test — it verifies Mockito wiring, not application behavior.
- **Extract repeated test data to named constants**: Any identifier, code, or string used in more than one test method should be declared as a `public static final`
  constant with a UUID-like value. This gives test data a semantic name and a single point of change.

<!-- 
The below guidelines may not be needed. Claude is trained on these.

- **Keep methods short and focused.** Long methods that do multiple things are hard to unit test. Extract logical units into separate, named methods.
- **Avoid `void` return types in business logic.** Methods that return values are directly assertable in tests. Prefer returning a result object over mutating state or producing side effects.
- **Design for testability.** Before writing a method, ask: can I call this in isolation and verify its output? If not, refactor the method's boundaries until you can.
-->

---

## Rule 1: General Test Guidelines
**ALWAYS:**
- Test edge cases and error conditions
- Test state transitions and business logic
- Name Tests for Behavior, Action, and Expected Result

**NEVER:**
- Test basic Java/library functionality (e.g., getters/setters, equals/hashCode unless custom logic)
- Test framework behavior (e.g., Spring's dependency injection)
- Test auto-generated code
- Use conditionals, loops, or complex expressions in tests
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
    
    long startTime = System.currentTimeMillis();
    
    // When
    AuthorizationResult result = service.authorize(card, amount);
    
    long executionTime = System.currentTimeMillis() - startTime;
    
    // Then
    assertThat(result.getStatus()).isEqualTo(AuthStatus.APPROVED);
    assertThat(executionTime)
        .as("Test should execute in less than 100ms, took: " + executionTime + "ms")
        .isLessThan(100L);
}

// ❌ BAD EXAMPLE: SLOW (ANTI-PATTERN) - Avoid this approach 
@Test
@Disabled("This test is too slow - makes real database and API calls")
void authorizePayment_SlowVersion() {
    // BAD: Creates real database connection
    DatabaseConnection db = new DatabaseConnection("jdbc:mysql://localhost:3306/payments");
    
    // BAD: Makes real HTTP call to payment gateway (takes 2-5 seconds)
    PaymentGateway realGateway = new VisaPaymentGateway("https://api.visa.com");
    
    // This test would take several seconds instead of milliseconds
    PaymentAuthorizationService service = new PaymentAuthorizationService(realGateway, db);
    // ... rest of test
}
```

***Examples of INDEPENDENT principle:***
```java
// ✅ GOOD EXAMPLE: INDEPENDENT: Each test sets up its own data and doesn't share state.
// Tests can run in any order without affecting each other.
class TransactionProcessorTest {
    @Test
    void transactionProcessing_processDebitCardWithSufficientFunds_returnsApproved() {
        // Given - Each test creates its own fresh instances
        TransactionProcessor processor = new TransactionProcessor();
        DebitCard debitCard = new DebitCard(
            "4532015112830366", 
            "12/25", 
            "123",
            new BigDecimal("1000.00") // Available balance
        );
        
        // When
        TransactionResult result = processor.process(
            debitCard, 
            new BigDecimal("50.00")
        );
        
        // Then
        assertThat(result.getStatus()).isEqualTo(TransactionStatus.APPROVED);
    }
    
    @Test
    void transactionProcessing_processDebitCardWithInsufficientFunds_returnsDeclined() {
        // Given - Independent setup, doesn't rely on previous test
        TransactionProcessor processor = new TransactionProcessor();
        DebitCard debitCard = new DebitCard(
            "4532015112830366", 
            "12/25", 
            "123",
            new BigDecimal("25.00") // Low balance
        );
        
        // When
        TransactionResult result = processor.process(
            debitCard, 
            new BigDecimal("50.00")
        );
        
        // Then
        assertThat(result.getStatus()).isEqualTo(TransactionStatus.DECLINED);
        assertThat(result.getDeclineReason()).isEqualTo("INSUFFICIENT_FUNDS");
    }
}

// ❌ BAD EXAMPLE: Tests that depend on each other (ANTI-PATTERN)
class BadTransactionProcessorTest {
    // Shared state - causes tests to be dependent
    private static Account sharedAccount;
    
    @Test
    @Order(1) // Test order matters - this is a red flag!
    void test1_CreateAccount() {
        sharedAccount = new Account(new BigDecimal("1000.00"));
        assertThat(sharedAccount.getBalance()).isEqualTo(new BigDecimal("1000.00"));
    }
    
    @Test
    @Order(2) // This test DEPENDS on test1 running first
    void test2_DeductFunds() {
        // PROBLEM: Fails if test1 doesn't run first
        sharedAccount.deduct(new BigDecimal("100.00"));
        assertThat(sharedAccount.getBalance()).isEqualTo(new BigDecimal("900.00"));
    }
}
```

***Examples of REPEATABLE principle:***
```java
// ✅ GOOD EXAMPLE: REPEATABLE: Test produces same results every time, regardless of environment.
// No dependency on current date, random values, or external systems.
class CurrencyConverterTest {
    @Test
    void feeCalculation_calculateForeignExchangeFeeRepeatedly_returnsSameResult() {
        // Given
        FeeCalculator calculator = new FeeCalculator();
        Money amount = Money.dollars(1000.00);
        
        // When - Run multiple times to prove repeatability
        BigDecimal fee1 = calculator.calculateForeignExchangeFee(amount);
        BigDecimal fee2 = calculator.calculateForeignExchangeFee(amount);
        BigDecimal fee3 = calculator.calculateForeignExchangeFee(amount);
        
        // Then - All results are identical
        assertThat(fee1).isEqualTo(fee2);
        assertThat(fee2).isEqualTo(fee3);
        assertThat(fee1).isEqualTo(new BigDecimal("30.00")); // 3% fee
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


// ❌ BAD EXAMPLE: - Not self-validating (ANTI-PATTERN)
class BadFraudDetectionTest {    
    @Test
    void fraudDetection_analyzeTransaction_writesToLogInsteadOfAsserting() {
        // Given
        FraudDetectionService fraudService = new FraudDetectionService();
        Transaction transaction = createTransaction();
        
        // When
        fraudService.analyze(transaction);
        
        // Then
        System.out.println("Check fraud_detection.log to see if fraud was detected"); // BAD: requires manual inspection
        // BAD: No assertions - test always passes
        // BAD: Developer must manually query the database to verify
        // "SELECT * FROM fraud_alerts WHERE transaction_id = ?"
    }
}
```

---

## Rule 3: Avoid Testing Implementation Details
**Rationale:** Tests using public APIs are resilient to refactoring and won't break when internal implementation changes.

**ALWAYS:**
- Test the "what" not the "how"
- Access the system under test the same way real users would
- Avoid testing private methods directly
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

// ❌ BAD EXAMPLE: Testing internal digit doubling logic (ANTI-PATTERN)
@Test
void cardValidation_doubleAlternateDigitsDirectly_returnsDoubledValues() {
    // Given - private implementation detail of Luhn algorithm
    CreditCardValidator validator = new CreditCardValidator();
    int[] digits = {4, 5, 3, 2, 0, 1, 5, 1};

    // When
    int[] doubled = validator.doubleAlternateDigits(digits);

    // Then
    assertThat(doubled).containsExactly(8, 5, 6, 2, 0, 1, 1, 1);
}

// ❌ BAD EXAMPLE: Testing internal card type detection logic (ANTI-PATTERN)
@Test
void cardValidation_detectCardTypeByBinDirectly_returnsVisa() {
    // Given - private method that identifies card type
    CreditCardValidator validator = new CreditCardValidator();

    // When
    CardType type = validator.detectCardType("4532015112830366");

    // Then
    assertThat(type).isEqualTo(CardType.VISA);
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

// ✅ GOOD EXAMPLE: Testing public API - each card network validated independently
@Test
void cardValidation_validateVisaCardNumber_returnsValid() {
    // Given
    CreditCardValidator validator = new CreditCardValidator();

    // When
    boolean result = validator.isValid("4532015112830366");

    // Then
    assertThat(result).isTrue();
}

@Test
void cardValidation_validateMastercardNumber_returnsValid() {
    // Given
    CreditCardValidator validator = new CreditCardValidator();

    // When
    boolean result = validator.isValid("5555555555554444");

    // Then
    assertThat(result).isTrue();
}

@Test
void cardValidation_validateAmexCardNumber_returnsValid() {
    // Given
    CreditCardValidator validator = new CreditCardValidator();

    // When
    boolean result = validator.isValid("378282246310005");

    // Then
    assertThat(result).isTrue();
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
- The test method name should have behavior, actions and expected outcomes
- Use descriptive names even if verbose

**NEVER:**
- Name tests vaguely or generically

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

    // ✅ GOOD EXAMPLE: Testing specific card type validation
    @Test
    void cardValidation_validateMastercardNumberWithValidFormat_returnsValid() {
        // Given
        CreditCardValidator validator = new CreditCardValidator();
        String validMastercard = "5555555555554444";

        // When
        boolean result = validator.isValid(validMastercard);

        // Then
        assertThat(result).isTrue();
    }
}
```

---

## Rule 5: Avoid Logic in Tests
**Rationale:** Tests should contain minimal logic; complex test logic indicates the test or production code needs refactoring.
**ALWAYS:**
- No conditionals (if/else) in test code
- No loops (for, while) in test code
- No complex expressions that obscure intent

**NEVER:**
- Use conditionals, loops, or complex expressions in tests

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
- Avoid shared mutable state between tests
- Keep setup methods focused and minimal
- Document non-obvious setup behavior

**NEVER:**
- Use shared mutable state between tests

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
    
    @AfterEach
    void tearDown() {
        // Clean up resources if needed
        // Mocks are automatically reset between tests
        processor = null;
        mockGateway = null;
        validCard = null;
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
    
    @AfterAll
    static void tearDownOnce() {
        // Clean up expensive resources
        networkRules = null;
        binDatabase = null;
    }
}
```

---

## Rule 7: Mock External Dependencies
**Rationale:** Use mocking frameworks to isolate the unit under test from external systems like AWS (Cloud) Services, databases, APIs, and file systems. This keeps tests fast, reliable, and prevents test failures due to external system issues.

**ALWAYS:**
- Mock external services only
  - Message queues/brokers (Kafka, SQS, SNS, etc.)
  - Cache systems (Redis, Memcached, ElastiCache)
  - Third-party libraries that make network calls (payment gateways, email services, etc.)
  - Databases (DynamoDB, Postgres, MySQL, etc.)
  - Cloud storage services (S3)
  - File systems
- Use frameworks like Mockito for Java
- Keep tests fast and reliable
- Prevent test failures due to external system issues

**NEVER:**
- Make real calls to external systems in unit tests
- Rely on external databases, APIs, or network resources
- Use real file system operations when mocking is appropriate

**Examples of Mocking External Dependencies**
```java
// ✅ GOOD EXAMPLE: Mocking external payment gateway API
@Test
void paymentProcessing_chargeCardViaMockedGateway_invokesGatewayCharge() {
    // Given
    PaymentGateway mockGateway = mock(PaymentGateway.class);
    when(mockGateway.charge(any(), any())).thenReturn(new GatewayResponse("SUCCESS"));
    PaymentService service = new PaymentService(mockGateway);
    CreditCard card = new CreditCard("4532015112830366", "12/25", "123");

    // When
    service.processPayment(card, new BigDecimal("99.99"));

    // Then
    verify(mockGateway, times(1)).charge(eq(card), eq(new BigDecimal("99.99")));
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

    // When - BAD: makes real network calls to AWS services
    // - Slow due to network latency
    // - Fails if AWS is down or credentials are invalid
    // - Costs money (AWS charges for DynamoDB/S3 operations)
    // - Pollutes production/test database with test payment records
    // - Stores unnecessary receipt files in S3
    processor.processPayment(request);

    // Then - no assertions; result depends on external state
}
```

---

## Rule 8: Use Interface-Based Fake Implementations for Stateful Complex External Dependencies
**Rationale:** For complex, stateful external service dependencies, prefer fake implementations over mocking frameworks. Fakes provide realistic behavior, are reusable across tests, and result in more maintainable test suites compared to mocks. When an external dependency is indirectly used (not directly injected), fake implementations provide a simpler and more maintainable testing approach.

**ALWAYS:**
- Prefer interface-based design for testability
- When refactoring is feasible, prefer fakes over mocks for better maintainability
- Use dependency injection to allow swapping real implementations with fakes during testing
- Fakes centralize implementation in one place; mocks scatter configuration across multiple test files

**NEVER:**
- Use mocking frameworks when production code cannot be changed (use Mockito for standard scenarios, PowerMock only as last resort)
- Make production code tightly coupled to concrete implementations

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
- Ignore exception testing for error conditions
- Use try-catch blocks in tests instead of assertThatThrownBy
- Test only success cases without validating failure scenarios

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

// ❌ BAD EXAMPLE: Using try-catch instead of assertThatThrownBy (ANTI-PATTERN)
@Test
void cardValidation_validateEmptyCardNumber_usesTryCatchInsteadOfAssertThatThrownBy() {
    // Given
    CreditCardValidator validator = new CreditCardValidator();

    // When - BAD: triggers the action inside a try block instead of an assertThatThrownBy lambda
    try {
        validator.isValid("");
        fail("Expected IllegalArgumentException to be thrown"); // BAD: manual failure marker
    } catch (IllegalArgumentException e) {
        // Then - BAD: verification mixed into the catch block, obscuring intent
        assertThat(e.getMessage()).isEqualTo("Card number cannot be empty");
    }
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
- Share mutable state across tests
- Depend on test execution order
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
- Test one behavior per test method

**NEVER:**
- Mix setup, action, and verification without clear separation
- Test multiple unrelated behaviors in a single test method
- Hide behavior-specific setup in @BeforeEach when it should be visible in the test

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
- Rely on default assertion messages like "expected: <true> but was: <false>"
- Write generic messages that don't explain the business context
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

// ❌ BAD EXAMPLE: Generic assertion with no context (ANTI-PATTERN)
@Test
void validateCard_blockedCard_unclearMessage() {
    // Given
    CreditCard card = new CreditCard("4532123456789012", "12/25", "123");
    card.setStatus(CardStatus.BLOCKED);
    card.setBlockedReason("FRAUD_SUSPECTED");
    CardValidator validator = new CardValidator();

    // When
    ValidationResult result = validator.validate(card);

    // Then
    // BAD: Failure message would be: "expected: ACTIVE but was: BLOCKED"
    assertThat(card.getStatus()).isEqualTo(CardStatus.ACTIVE);
}

// ✅ GOOD EXAMPLE: Descriptive message with object context
@Test
void validateCard_blockedCard_clearMessage() {
    // Given
    CreditCard card = new CreditCard("4532123456789012", "12/25", "123");
    card.setStatus(CardStatus.BLOCKED);
    card.setBlockedReason("FRAUD_SUSPECTED");
    CardValidator validator = new CardValidator();

    // When
    ValidationResult result = validator.validate(card);

    // Then
    // GOOD: Failure message includes full context:
    // "Expected card state ACTIVE, but got card <{number: '****9012', state: 'BLOCKED', blockedReason: 'FRAUD_SUSPECTED'}>"
    assertThat(card.getStatus())
        .as("Expected card state ACTIVE, but got card <{number: '%s', state: '%s', blockedReason: '%s'}>",
            card.getMaskedNumber(), card.getStatus(), card.getBlockedReason())
        .isEqualTo(CardStatus.ACTIVE);
}
```

---

## Final Step: Verify Compilation

After writing all test methods, compile the test source to confirm there are no errors.
Fix any compilation errors before completing — do not leave broken code.
Repeat until the build is clean.
