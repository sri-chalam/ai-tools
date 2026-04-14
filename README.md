# AI Skills for Java & Cloud Development

A collection of AI instruction skills for GitHub Copilot, designed to enforce best practices across Java, AWS, and Cloud-native development workflows.

These skills are structured as GitHub Copilot Agent Skills — each skill lives in `.github/skills/` and is automatically activated by Copilot when working on relevant files.

---

## Available Skills

### ✅ JUnit Best Practices
**Location:** `.github/skills/junit-best-practices/`
**Applies to:** `**/*Test.java`, `**/*Tests.java`, `**/*Spec.java`

Guides writing production-quality JUnit 5 unit tests in Java. Covers:
- FIRST principles (Fast, Independent, Repeatable, Self-validating, Timely)
- Given-When-Then (GWT) test structure
- Descriptive test naming conventions
- Mocking and faking strategies
- Exception testing
- Avoiding logic and shared mutable state in tests

---

## Repository Structure

```
.
├── .github/
│   ├── copilot-instructions.md       # Top-level Copilot pointer
│   └── skills/
│       └── junit-best-practices/
│           ├── skill.md
│           ├── instructions.md
│           └── scripts/
└── README.md
```
