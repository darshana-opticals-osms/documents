# OSMS Coding and Code Review Standards

## 1. Purpose

This document defines the coding and code review standards for the Optical Shop Management System (OSMS) development team.

The purpose of these standards is to ensure that all team members write code that is consistent, readable, maintainable, secure, testable, and easy to review.

These standards apply to all contributors working on the OSMS frontend, backend, infrastructure, and supporting development files. They also define the team's expectations for coding practices, naming, formatting, testing, Git usage, pull requests, and code reviews.

---

## 2. General Coding Principles

All OSMS code must follow the engineering principles defined in the SENG 34213 development guideline.

### 2.1 SOLID Principles

The team should follow the SOLID principles where applicable:

- **Single Responsibility Principle** – A module, class, component, or function should have one clear responsibility.
- **Open/Closed Principle** – Code should be designed so that behaviour can be extended without unnecessarily modifying existing stable code.
- **Liskov Substitution Principle** – Derived implementations should remain compatible with the behaviour expected from their base abstractions.
- **Interface Segregation Principle** – Components should depend only on interfaces or functionality they actually require.
- **Dependency Inversion Principle** – High-level logic should not be tightly coupled to low-level implementation details.

### 2.2 DRY – Don't Repeat Yourself

Duplicated logic should be avoided.

Reusable logic should be extracted into appropriate shared modules such as:

- Utility functions
- Services
- Hooks
- Reusable components
- Middleware

The same business rule should not be implemented independently in multiple locations.

### 2.3 YAGNI – You Aren't Gonna Need It

Only functionality required by the current issue, sprint, and acceptance criteria should be implemented.

Developers should avoid adding speculative features that are not currently required.

### 2.4 Clean Code

Code should:

- Use meaningful and descriptive names.
- Use small and focused functions.
- Keep functions and modules focused on one responsibility.
- Minimize unnecessary side effects.
- Avoid duplicated logic.
- Avoid unexplained magic numbers or magic strings.
- Remain understandable to another team member without requiring explanation from the original author.

### 2.5 Separation of Concerns

Presentation, business logic, data access, configuration, validation, and other responsibilities should remain appropriately separated.

Frontend UI components should not contain unnecessary backend or data-access logic, and backend route handlers should not contain unnecessary persistence or business logic directly.

---

## 3. JavaScript / TypeScript Standards

OSMS uses a JavaScript-based MERN technology stack.

The team adopts the following development tools and style approach:

- **Airbnb JavaScript Style Guide** as the general JavaScript/TypeScript style reference.
- **ESLint** for automated code-quality and linting checks.
- **Prettier** for consistent code formatting.

The ESLint and Prettier configuration used by each code repository must be committed to that repository so that every team member follows the same rules.

Developers must not disable linting rules simply to hide valid errors.

Before submitting a pull request, developers should ensure that:

- The code passes ESLint.
- The code passes the configured Prettier formatting check.
- No new linting errors are introduced.
- Unused variables and unnecessary code are removed.
- Debugging statements such as unnecessary `console.log()` calls are removed before merge unless intentionally required.

---

## 4. Naming Conventions

Consistent naming must be used throughout the OSMS codebase.

### 4.1 Variables and Functions

Use **camelCase** for variables and functions.

Examples:

```javascript
customerName
getCustomerProfile()
calculateLoyaltyPoints()
```

Names should clearly communicate their purpose.

Avoid unclear names such as:

```javascript
x
data1
temp2
abc
```

unless the context clearly justifies a short name.

### 4.2 Classes and React Components

Use **PascalCase** for classes and React components.

Examples:

```javascript
CustomerService
InventoryManager
ProductCard
LoginPage
```

### 4.3 Constants

Use **UPPER_SNAKE_CASE** for constants that represent fixed shared values.

Examples:

```javascript
MAX_LOGIN_ATTEMPTS
DEFAULT_PAGE_SIZE
```

### 4.4 Boolean Values

Boolean names should clearly describe a true/false condition.

Preferred examples:

```javascript
isAuthenticated
hasPermission
canEditInventory
```

### 4.5 Files

React component files should use meaningful PascalCase names where appropriate.

Examples:

```text
ProductCard.jsx
LoginPage.jsx
AppointmentForm.jsx
```

Other JavaScript/TypeScript files should use consistent descriptive names.

Examples:

```text
authService.js
userController.js
errorHandler.js
```

### 4.6 Branches

Branch names must follow the development guideline.

```text
feature/<ticket-id>-<slug>
fix/<ticket-id>-<slug>
hotfix/<slug>
release/<version>
```

For OSMS, the numeric DDP ticket identifier should be used as the ticket ID.

Example:

```text
DDP-020
```

becomes:

```text
feature/20-coding-standards
```

Branch slugs must be:

- Lowercase
- Short but descriptive
- Separated using hyphens

---

## 5. Project Structure Guidelines

The project structure must support the approved OSMS architecture and separation of concerns.

### 5.1 Backend

Backend code should clearly separate responsibilities such as:

- Routes
- Controllers
- Services / business logic
- Models / data layer
- Middleware
- Configuration
- Utilities
- Tests

Business logic should not be unnecessarily placed directly inside route definitions.

Database operations, request handling, and business rules should remain appropriately separated.

### 5.2 Frontend

Frontend code should separate responsibilities such as:

- Pages
- Reusable components
- Services / API communication
- State management
- Hooks
- Routing
- Utilities
- Tests

API communication should not be unnecessarily duplicated across UI components.

Reusable UI elements should be implemented as reusable components instead of repeating the same markup and logic.

### 5.3 General Rules

- Keep modules focused on one responsibility.
- Avoid large files containing unrelated functionality.
- Reuse shared functionality instead of duplicating it.
- Follow the architecture defined in the approved SDS.
- Significant deviations from the approved architecture must be documented through an Architectural Decision Record (ADR).

---

## 6. Error Handling

Errors must be handled clearly, consistently, and securely.

Developers must:

- Handle expected error conditions.
- Return or display meaningful error information where appropriate.
- Avoid silently ignoring errors.
- Avoid exposing sensitive internal information.
- Ensure asynchronous errors are handled correctly.
- Avoid exposing stack traces to users in production.
- Keep error-handling responsibilities separate from unrelated business logic where possible.

Backend APIs should use appropriate HTTP status codes.

Examples include:

- `400` – Bad Request
- `401` – Unauthorized
- `403` – Forbidden
- `404` – Not Found
- `409` – Conflict
- `500` – Internal Server Error

Reviewers must verify that error cases are properly handled and that failures are communicated meaningfully.

---

## 7. Security Practices

Security requirements apply to all OSMS development work.

### 7.1 Secrets

Secrets must never be committed to Git.

This includes:

- Passwords
- API keys
- JWT secrets
- Authentication tokens
- Database connection strings
- Production credentials

Local secrets must be stored in `.env` files that are excluded using `.gitignore`.

A `.env.example` file containing required variable names and placeholder values should be committed where environment configuration is required.

GitHub Secrets must be used for CI/CD secrets.

### 7.2 Authentication and Authorization

- Passwords must never be stored as plain text.
- Passwords must use secure bcrypt hashing according to the project security requirements.
- Protected backend functionality must enforce authorization on the server.
- Role-based access control must not rely only on hiding frontend controls.
- Unauthorized access must be tested.

### 7.3 Input Protection

All external input must be validated and sanitized before being trusted.

Developers must protect the application against risks such as:

- Injection attacks
- Invalid input
- Broken access control
- Security misconfiguration

### 7.4 Logging and Errors

Logs and error messages must not contain:

- Passwords
- Tokens
- Secrets
- Database credentials
- Unnecessary personally identifiable information
- Sensitive clinical information

Production error responses must not expose internal stack traces or implementation details.

### 7.5 Dependencies

Project dependencies must be checked for known vulnerabilities.

No known high or critical dependency vulnerability should remain unresolved before release.

---

## 8. Testing Expectations

Testing is part of the Definition of Done.

Code without the required tests must not be considered complete.

### 8.1 Unit Tests

Unit tests should test individual functions, classes, services, or components in isolation.

The project target is:

```text
At least 80% coverage of new code
```

Unit tests should follow the **Arrange – Act – Assert (AAA)** pattern.

Test descriptions should clearly communicate behaviour using a Given-When-Then style where appropriate.

### 8.2 Integration Tests

Integration tests must verify communication between modules and services.

Backend API endpoints must have appropriate integration tests.

Where database integration is required, tests should use a test database with controlled test data.

### 8.3 End-to-End Tests

End-to-end testing should verify complete user journeys through the frontend and backend.

E2E testing should cover:

- Important happy paths.
- The project's top critical user flows.

### 8.4 Test Quality

Tests must:

- Verify behaviour rather than unnecessary implementation details.
- Include positive cases.
- Include relevant negative and error cases.
- Be repeatable.
- Use controlled test data.
- Remain independent where practical.

All required tests must pass before a pull request is merged.

---

## 9. Git / Commit Conventions

OSMS follows the Git branching strategy and Conventional Commits format defined by the SENG 34213 development guideline.

### 9.1 Main Branches

#### `main`

Contains production-ready code.

Changes must reach `main` through the approved pull request and release process.

#### `develop`

Acts as the integration branch.

Feature and normal fix branches are merged into `develop` before release.

### 9.2 Working Branches

```text
feature/<ticket-id>-<slug>
fix/<ticket-id>-<slug>
hotfix/<slug>
release/<version>
```

One working branch should be used per GitHub development issue.

Feature branches should remain short-lived.

### 9.3 Conventional Commit Types

Use the following commit types:

| Type | Purpose |
| --- | --- |
| `feat` | New feature |
| `fix` | Bug fix |
| `test` | Add or update tests |
| `ci` | CI/CD configuration |
| `build` | Build system or dependency change |
| `perf` | Performance improvement |
| `refactor` | Code restructuring without changing behaviour |
| `docs` | Documentation update |
| `chore` | Tooling, configuration, or housekeeping |
| `style` | Formatting-only change |
| `revert` | Revert a previous commit |

Commit messages should follow this structure:

```text
type(scope): short description
```

Example:

```text
feat(auth): implement user login
```

Documentation example:

```text
docs(standards): define team coding and review standards
```

Commit messages must be clear and describe the change performed.

The related GitHub issue must be linked where required using:

```text
Closes #<issue-number>
```

Example for this documents repository issue:

```text
Closes #4
```

---

## 10. Pull Request and Code Review Standards

Every pull request must receive at least **one approving peer review** before merging.

Normal feature work must be submitted to the `develop` branch through a pull request.

### 10.1 Reviewer Responsibilities

Reviewers must evaluate the following areas.

#### Correctness

- Does the implementation satisfy the issue requirements?
- Are all acceptance criteria satisfied?
- Do the required tests pass?

#### Readability

- Can another team member understand the implementation?
- Are names clear and meaningful?
- Is unnecessary complexity avoided?

#### Architecture

- Does the implementation follow the approved SDS and agreed architecture?
- Are responsibilities appropriately separated?

#### Test Quality

- Are meaningful tests included?
- Do tests verify behaviour?
- Are relevant negative cases covered?

#### Security

Reviewers should check for issues such as:

- Hardcoded secrets
- Missing authorization
- Unsafe input handling
- Injection risks
- Exposure of sensitive data

#### Performance

Reviewers should identify clearly inefficient implementations such as unnecessary repeated queries or avoidable blocking operations.

#### Error Handling

Reviewers must verify that relevant error cases are handled and communicated appropriately.

### 10.2 Review Comment Convention

Review comments must use the following prefixes:

- **`[blocker]`** – Must be resolved before the PR can be merged.
- **`[suggestion]`** – Recommended improvement, but not mandatory for the current PR.
- **`[question]`** – Request for clarification; does not automatically indicate an error.
- **`[nit]`** – Minor style or preference comment.

Example:

```text
[blocker] User input is being stored without server-side validation.
```

The author must not resolve a reviewer's comment on behalf of the reviewer.

After the author makes the required change, the reviewer should verify the change and resolve the comment.

### 10.3 Pull Request Quality Rules

A pull request must not be considered complete until:

- Required acceptance criteria are satisfied.
- Required tests pass.
- Linting passes.
- Formatting requirements are satisfied.
- CI checks pass.
- At least one peer review is completed.
- All `[blocker]` comments are resolved.
- No secrets are committed.
- Required documentation is updated.

---

## 11. Definition of Done

The following Definition of Done applies to development work where relevant.

Before an issue is considered complete:

- [ ] All Acceptance Criteria are verified.
- [ ] Code follows the agreed team style guide.
- [ ] Linting passes with no new errors.
- [ ] Unit tests are written for new logic and pass.
- [ ] Integration tests are written for new API endpoints and pass.
- [ ] New code achieves at least 80% required coverage.
- [ ] No hardcoded secrets or credentials are present.
- [ ] A pull request is raised against `develop`.
- [ ] The pull request description is completed.
- [ ] At least one peer review is completed.
- [ ] All `[blocker]` review comments are resolved.
- [ ] CI checks pass.
- [ ] The completed feature is smoke-tested in staging where applicable.
- [ ] API documentation is updated when an endpoint is added or changed.
- [ ] `CHANGELOG.md` is updated under `[Unreleased]` where applicable.
- [ ] The related GitHub issue is linked using `Closes #<issue-number>`.
- [ ] The GitHub issue is moved to `Done` after successful merge.

These standards must be followed by all OSMS development team members throughout the development phase.