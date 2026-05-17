# JUnit Best Practices Skill (Java)

An AI skill that enforces production-quality JUnit 5 test standards for Java projects.

## Summary

This skill activates comprehensive JUnit 5 best practices whenever you write, review, or modify Java test code. It is automatically applied to test source files matching standard Java test naming conventions.

**Key Features:**
- Enforces FIRST principles (Fast, Independent, Repeatable, Self-Validating, Timely)
- Promotes behavior-driven test naming: `behavior_action_expectedResult`
- Requires Given-When-Then (GWT) structure for all tests
- Distinguishes when to use Mockito mocks vs. interface-based fakes
- Covers exception testing, setup annotation usage, and descriptive failure messages
- Prevents common anti-patterns: logic in tests, implementation detail testing, shared mutable state

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
| 9 | Test expected exceptions with `assertThrows` |
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
git clone https://github.com/sri-chalam/ai-tools.git
```

**Step 2: Create a symbolic link**

```bash
ln -s /path/to/ai-tools/skills/in-progress/junit-best-practices ~/.claude/skills/junit-best-practices
```

Replace `/path/to/ai-tools` with the actual path where you cloned the repository.

**Updating the skill**

```bash
cd /path/to/ai-tools
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

---

## Files

- `SKILL.md` — Skill metadata and `applyTo` configuration
- `instructions.md` — Full rule definitions with good and bad examples for each rule
