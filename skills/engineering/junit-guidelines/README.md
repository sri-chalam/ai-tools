# JUnit Guidelines Skill

An AI skill that enforces production-quality JUnit 5 test standards for Java projects.

## Summary

This skill activates comprehensive JUnit 5 guidelines whenever you write, review, or modify Java test code. It is automatically applied to test source files matching standard Java test naming conventions.

**Key Features:**
- Enforces FIRST principles (Fast, Independent, Repeatable, Self-Validating, Timely)
- Promotes behavior-driven test naming: `behavior_action_expectedResult`
- Requires Given-When-Then (GWT) structure for all tests
- Distinguishes when to use Mockito mocks vs. interface-based fakes
- Covers exception testing, setup annotation usage, and descriptive failure messages
- Prevents common anti-patterns: logic in tests, implementation detail testing, shared mutable state

**Core Testing Philosophy:**

Before the numbered rules, the skill establishes these foundational principles:

- **Fast, Deterministic, Isolated, Maintainable, Clear** — the five qualities every test must have
- **Tests as specifications** — tests document what the system must do, forming a contract that persists across refactorings
- **Test observable behavior through public APIs, not implementation details** — well-written tests survive internal refactors
- **Validate a specific behavior or outcome** — each test represents one complete scenario with a clear expected result
- **A test that cannot catch a real bug should not be written** — if a method has no conditional logic, transformation, or error handling and only forwards arguments to a dependency, skip the test; it verifies Mockito wiring, not application behavior
- **Extract repeated test data to named constants** — any identifier, code, or string used in more than one test should be a `public static final` constant, giving it a semantic name and a single point of change

**Rules Covered:**

| Rule | Summary |
|------|---------|
| 1 | General test guidelines — what to test and what to skip |
| 2 | FIRST principles — Fast, Independent, Repeatable, Self-Validating, Timely |
| 3 | Avoid testing implementation details — test public APIs, not private internals |
| 4 | Descriptive test names — behavior, action, and expected result |
| 5 | No logic in tests — no conditionals, loops, or complex expressions |
| 6 | Use `@BeforeEach` / `@BeforeAll` appropriately |
| 7 | Mock external dependencies (databases, cloud services, APIs) |
| 8 | Use interface-based fakes for stateful, complex dependencies |
| 9 | Test expected exceptions with `assertThatThrownBy` |
| 10 | Keep tests independent — no shared mutable state |
| 11 | Given-When-Then structure for readability |
| 12 | Write descriptive failure messages |

---

## How to Use

This skill auto-applies when working in a Claude Code session that has the skill loaded. No manual invocation is needed — the instructions activate automatically when Claude opens, generates, or reviews files matching:

- `src/test/**/*.java`
- `**/*Test.java`
- `**/*Tests.java`

**Step 1: Clone the repository**

```bash
# Public Soruce
git clone https://github.com/sri-chalam/ai-tools.git

# Any other repo that is hosting this skill
git clone <URL of the repository hosting this skill>
```

**Step 2: Create a symbolic link**

```bash
ln -s /path/to/skill-directory/junit-guidelines ~/.claude/skills/junit-guidelines
```

Replace `/path/to/skill-directory` with the actual path where you cloned the repository.

**Updating the skill**

```bash
cd /path/to/skill-directory
git pull
```

Then restart your AI agent (e.g., restart VS Code) to pick up the latest changes.

---

## Why This Skill?

### Tests as Specifications

Tests should document what the system must do, not how it does it. When internal logic is refactored, well-written tests remain stable because they verify outcomes through public APIs.

### Common Problems This Skill Prevents

- Tests that break during refactoring because they test private methods or internal state
- Slow tests that make real database or network calls
- Shared mutable state causing test-order dependencies
- Vague test names that don't communicate the failure scenario
- Missing exception tests that leave error paths uncovered
- Overuse of mocking frameworks where interface-based fakes would be cleaner and more maintainable
- Tests that verify Mockito wiring rather than application behavior (e.g., pure-delegation methods with no conditional logic)
- Magic strings and IDs scattered across multiple test methods with no single point of change

---

## Files

- `SKILL.md` — Skill metadata, `applyTo` configuration, and full rule definitions with good and bad examples for each rule
