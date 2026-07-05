# JUnit Guidelines Skill

An AI skill that generates and reviews JUnit 5 tests following best practices for Java projects.

## Summary

Poorly guided AI-generated tests can do more harm than good during refactoring — see [Why AI-Generated Tests Need Explicit Guidance](#why-ai-generated-tests-need-explicit-guidance) for details.

This skill applies established unit test best practices to JUnit 5, activated whenever Java unit test code is written, reviewed, or modified.

## Common Use Cases
- Generate tests for uncommitted changes or recent commits
- Generate tests for a specific class
- Fill gaps in an existing test class
- Survey the codebase for test-coverage ROI
- Verify a PR's logic changes have well-written tests

See [Sample Prompts for Manual Invocation](#sample-prompts-for-manual-invocation) for ready-to-use prompts.

## Key Features
- Enforces FIRST principles (Fast, Independent, Repeatable, Self-Validating, Timely)
- Promotes behavior-driven test naming: `behavior_action_expectedResult`
- Requires Given-When-Then (GWT) structure for all tests
- Distinguishes when to use Mockito mocks vs. interface-based fakes
- Distinguishes logic-owning methods (test exhaustively) from orchestrating methods (test representative wiring only)
- Covers exception testing, setup annotation usage, and descriptive failure messages
- Prevents common anti-patterns: logic in tests, implementation detail testing, shared mutable state

## Core Testing Philosophy

Before the numbered rules, the skill establishes these foundational principles:

- **Fast, Deterministic, Isolated, Maintainable, Clear** — the five qualities every test must have
- **Tests as specifications** — tests document what the system must do, forming a contract that persists across refactorings
- **Test observable behavior through public APIs, not implementation details** — well-written tests survive internal refactors
- **Validate a specific behavior or outcome** — each test represents one complete scenario with a clear expected result
- **A test that cannot catch a real bug should not be written** — if a method has no conditional logic, transformation, or error handling and only forwards arguments to a dependency, skip the test; it verifies Mockito wiring, not application behavior
- **Extract repeated test data to named constants** — any identifier, code, or string used in more than one test should be a `public static final` constant, giving it a semantic name and a single point of change

## Rules Covered

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
| 13 | Test logic-owning methods exhaustively; test orchestrating methods for wiring only |

## AI Agent Compatibility

This skill's guidelines are written for any AI coding agent (Claude Code, Codex, Copilot, etc.). However, the final validation step relies on a subagent — a fresh-context reviewer that checks generated tests against the guidelines, fixes discrepancies, and resolves compile issues. A subagent is used so validation happens with a fresh pair of eyes rather than the same context that wrote the tests, but the subagent invocation mechanism itself is Claude Code-specific and its instructions may not work in other AI coding agents. There does not appear to be a portable, cross-agent way to invoke subagents at this time. On other agents, test generation still follows the guidelines directly; only the separate validation pass is unavailable.

---

## How to Use

This skill can be invoked two ways. It auto-applies when working in an AI coding agent session that has the skill loaded (e.g., Claude Code) and the session context matches the skill's description — unit testing Java classes — activating automatically when the agent opens, generates, or reviews files matching:

- `src/test/**/*.java`
- `**/*Test.java`
- `**/*Tests.java`

It can also be invoked manually with `/junit-guidelines`, regardless of context. Most of the example prompts below use this manual invocation form.

**Step 1: Clone the repository**

```bash
# Public Soruce
git clone https://github.com/sri-chalam/ai-tools.git

# Any other repo that is hosting this skill
git clone <URL of the repository hosting this skill>
```

**Step 2: Create a symbolic link**

```bash
ln -s /path/to/ai-tools/skills/engineering/junit-guidelines ~/.claude/skills/junit-guidelines
```

Replace `/path/to/ai-tools` with the actual path where you cloned the repository.

> **Note:** The skill is installed at the user level (`~/.claude/skills/`) so it is available across all your projects. A symbolic link is used instead of copying files because the skill evolves frequently — a `git pull` in the cloned repository immediately picks up the latest guidelines without re-copying anything.

> **Note:** The path `~/.claude/skills/` is specific to Claude Code. If you are using a different AI coding assistant, point the symbolic link to the equivalent user-level skills directory for that tool.

**Step 3: Symlink the validator subagent**

The skill's final validation step delegates to the `junit-validator` subagent, which Claude Code resolves from `~/.claude/agents/`. The subagent ships in the repo at `agents/engineering/junit-guidelines/junit-validator.md`, so link it into place:

```bash
ln -s /path/to/ai-tools/agents/engineering/junit-guidelines/junit-validator.md ~/.claude/agents/junit-validator.md
```

Replace `/path/to/ai-tools` with the actual path where you cloned the repository.

> **Note:** As with the skill, a symbolic link (not a copy) means a `git pull` immediately picks up validator updates. If `~/.claude/agents/` does not exist yet, create it first with `mkdir -p ~/.claude/agents`.

**Updating the skill**

```bash
cd /path/to/ai-tools
git pull
```

A single `git pull` refreshes both symlinks — the skill and the validator subagent — since they point into the same clone.

Then restart your AI agent (e.g., restart VS Code) to pick up the latest changes.

**After Setup**

After adding the symbolic link and restarting, whenever there is a need to work with test cases, the model usually loads the Skill automatically. Sometimes the model may skip loading the skill if the context does not clearly indicate test-related work.

### Sample Prompts for Manual Invocation

Explicitly invoking the skill guarantees it is loaded, regardless of context. The following are example prompts you can copy, adapt, and paste into AI coding agent:

Generate tests for local changes that aren't committed yet (staged and unstaged):
```bash
/junit-guidelines Using the JUnit guidelines, review my uncommitted changes (`git diff HEAD`) and generate or update JUnit tests covering the new/modified behavior.
```

Generate tests for changes introduced by the last two commits:
```bash
/junit-guidelines Using the JUnit guidelines, review the changes introduced by the last two commits (`git diff HEAD~2 HEAD`) and generate or update JUnit tests covering the new/modified behavior.
```

Generate tests for a specific file:
```bash
/junit-guidelines Using the JUnit guidelines, generate test cases for @/path/to/OrderService.java
```

Find and fill gaps in an existing test class:
```bash
/junit-guidelines Using the JUnit guidelines, review @/path/to/OrderService.java and its existing tests in @/path/to/OrderServiceTest.java. Identify missing test coverage (conditional logic paths, edge cases, exceptions) and add the missing test methods.
```

Survey the codebase and rank classes by test-writing ROI (report only, no code changes):
```bash
/junit-guidelines Using the JUnit guidelines, explore the codebase and identify classes whose existing tests (or lack thereof) fall short of the guidelines. Report the top 10 classes that would benefit most from added test coverage, ranked by expected benefit, with a one-line reason for each (e.g. untested logic-owning methods, missing exception paths, no tests at all). Do not write any test code yet — just the report.
```

Verify a PR's logic changes have well-written tests (report only, no code changes):

> `gh pr diff` alone only shows diff hunks, no surrounding context. Prefer checking out the branch first (`git switch <source-branch>`, or `gh pr checkout 123` if you only have the PR number) so the model can read full files.

```bash
/junit-guidelines Using the JUnit guidelines, review this PR's diff (`gh pr diff 123`). For each new or changed logic-owning method, check whether the PR's tests actually exercise that logic (conditional logic paths, edge cases, exceptions) — not just whether a test method exists. Report any business logic added or changed without corresponding test coverage. Do not write any test code yet — just the report.
```

Find test classes that should use fakes instead of mocks (report only, no code changes):
```bash
/junit-guidelines Using the JUnit guidelines, explore the codebase's test classes and identify those mocking stateful, complex external dependencies where an interface-based fake would be more appropriate. Report the top 10 test classes ranked by expected maintainability benefit, with a one-line reason for each (e.g. dependency has multi-step stateful behavior, mock setup is duplicated across many tests). Do not write any code yet — just the report.
```

---

## Why This Skill?

### Why AI-Generated Tests Need Explicit Guidance

Writing unit tests is easy for an AI coding agent — too easy. **Without clear guidance, an agent will happily generate large volumes of tests that mock everything and verify nothing meaningful — commonly called "slop" tests: code that compiles and passes but adds no real verification value**. Two examples of this show up in opposite directions: over-mocked tests silently fail to catch real regressions, while tests coupled to private implementation details break noisily on every refactor even when behavior is unchanged. Both erode trust in the suite — one through false confidence, the other through false alarms. That's worse than having no tests: when the application is later refactored, a wall of low-value tests breaks alongside (or instead of) real regressions, and developers can no longer tell whether a failure signals an actual bug or just a brittle, badly written test. This skill exists to collate best practices — with good and bad examples — so AI-generated tests stay few, meaningful, and genuinely useful during refactoring rather than noise to wade through.

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

- `SKILL.md` — Skill metadata and full rule definitions with good and bad examples for each rule
- `agents/engineering/junit-guidelines/junit-validator.md` — The read-only `junit-validator` subagent invoked by the skill's final step to validate generated tests in a fresh context and return a findings table
