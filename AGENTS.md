# TypeScript Development Guidelines

## Core Principles

- Use the simplest possible solution
- Prefer procedural programming and functional programming paradigms
- **Simplicity**: Prioritize **Cognitive Simplicity** over **Code Brevity**. Explicit is better than implicit. Unnested code is better than nested. Code that can be read top-to-bottom is better than clever abstractions. Service Oriented Architecture is intended for COMPLEX projects. For simple tasks, keep it simple.
- **Conflict Resolution**: If "Simplicity" conflicts with "Robustness" (e.g., Error Handling or SRP), **Robustness wins**. It is better to have verbose, safe, and testable code than concise but fragile code.
- **Avoid Classes**: Prefer object literals returned by factory functions, along with TypeScript interfaces.
- Use well-maintained, mature libraries where possible (check npm weekly downloads and last update date)
- Ensure you include JSDoc comments for all exported functions, classes, and interfaces
- Run the linter (ESLint) as frequently as possible to detect any issues and fix them
- Avoid overriding the linter or using language escape hatches such as:
  - `any` type assertions
  - `@ts-ignore` or `@ts-expect-error` comments
  - Type assertions without proper validation
- After you have finished your implementation:
  - Run the TypeScript compiler (`tsc`) to check for type errors
  - Test your implementation using the project's test framework (Jest, Vitest, etc.)
  - If you don't know how to test your implementation, ask the user for help
- When handling unknown data structures, use a validation library such as Zod, io-ts, or yup

## Pre-Implementation Design Protocol

**Analysis & Planning**: For architectural changes, new services, or complex logic, you MUST first output a brief plan. This plan serves as the basis for the **Design Review** you conduct with the user. It should explicitly address:
1.  **Safety**: Identify potential edge cases or error states.
2.  **Architecture**: Verify if Single Responsibility Principle (SRP) is maintained. If a "mode" or variation is detected, explicitly list the strategy objects/functions to be created.
3.  **Result Definition**: Define the specific `Result<T, E>` types that will be returned, ensuring errors are strongly typed.
4.  **External API Exploration**: Before implementing external API integrations, manually test the API (e.g., using curl) to understand actual response shapes and document them.

## Architecture & Design

- **Single Responsibility Principle**: Each module, class, or function should have one, and only one, reason to change. Decompose large, multi-purpose components into smaller, focused ones.
- **Composition over Configuration/Inheritance**: Favor composition to define behavior. If a component operates in different "modes" with overlapping but distinct dependencies or logic, do not use internal flags or configuration options to switch behavior. Instead, extract shared dependencies and create distinct implementations (strategies) for each mode.
- **Service Oriented Architecture**: For **COMPLEX** projects, decompose systems into small, individually testable services located under a `lib` folder.
- **Implementation Style**: When implementing services, prefer a **procedural** or **functional** style over complex object-oriented patterns.
- **Dependency Structure**: Design services with orthogonality, layering, and proper abstraction levels in mind. Tend towards a tree or diamond dependency pattern. Apply **SOLID principles**.
- **Adapter Pattern**: Always wrap external APIs or services in an adapter service with minimal logic.
- **Design Review**: You MUST always have the user review your service design before implementation.
- **Test Confirmation**: You MUST always ask the user if they would like tests to be added when performing a task.
- **Manual Testing**: If the user declines automated tests, you MUST try to test manually. If you don't know how to test manually, you MUST ask the user for help.

## Anti-patterns

- **Mode Flags**: Avoid functions or components that take a boolean flag or "mode" enum to significantly alter their control flow or dependencies.
  - **Exception**: Simple configuration flags that do not drastically change logic (e.g., `verbose: true`, `dry_run: true`) are acceptable.
  - **Bad**:
    ```typescript
    function processUser(user: User, mode: 'full' | 'lite') {
       if (mode === 'full') { /* ... */ }
    }
    ```
  - **Good**: **Strongly Prefer** the Strategy Pattern by creating separate functions (`processUserFull`, `processUserLite`) or injecting a strategy object that handles the processing variation.
- **Testing Against Mental Model**: Avoid writing tests that validate assumptions rather than real API contracts. Tests that pass based on expected behavior but fail against actual external system responses indicate this anti-pattern.
  - **Remediation**: Test adapters with real API calls whenever practical. When real calls are impractical, use fixture-captured real responses. Unit test all string manipulation (URL construction, ID parsing).
- **Dead Code**: Avoid leaving unused code in the codebase, including unused imports, unreachable code paths, commented-out code, unused variables, unused functions, and unused types. Dead code increases cognitive load, obscures intent, and can mislead future readers.
  - **Remediation**: Use ESLint rules (`@typescript-eslint/no-unused-vars`, `no-unreachable`) and tools like `ts-prune` to detect dead code. Remove it immediately rather than commenting it out. Version control preserves history if code needs to be recovered later.

## TypeScript-Specific Guidelines

### Functions and Methods

- **Context Objects**: When functions share common context or state (e.g., database client, logger, external service, callbacks, or common data parameters), use a "context object" passed as the first argument with an explicit interface:

  ```typescript
  interface ServiceContext {
    db: DatabaseClient;
    logger: Logger;
    config: AppConfig;
  }

  function getUser(ctx: ServiceContext, userId: string): Promise<User> {
    ctx.logger.info("Fetching user", { userId });
    return ctx.db.users.findById(userId);
  }

  function updateUser(
    ctx: ServiceContext,
    userId: string,
    data: UserUpdate,
  ): Promise<User> {
    ctx.logger.info("Updating user", { userId });
    return ctx.db.users.update(userId, data);
  }
  ```

- **Avoid Destructuring in Function Arguments**: Do not destructure objects in function parameters. Instead, accept the object as-is and destructure at the top of the function body. This keeps function signatures and JSDoc cleaner, and makes it easy to pass the object to nested function calls:

  ```typescript
  // Avoid this:
  function processOrder({ orderId, items, customer }: OrderInput): Result {
    // ...
  }

  // Prefer this:
  function processOrder(input: OrderInput): Result {
    const { orderId, items, customer } = input;
    // Now 'input' can easily be passed to helper functions
    validateOrder(input);
    // ...
  }
  ```

- Provide JSDoc comments with examples:
  ```typescript
  /**
   * Calculates the total price including tax
   * @param price - Base price in cents
   * @param taxRate - Tax rate as a decimal (e.g., 0.08 for 8%)
   * @returns Total price in cents
   * @example
   * calculateTotal(1000, 0.08) // returns 1080
   */
  ```

### Error Handling

- **Favor Result Objects over Exceptions**: For expected domain errors, return a structured result object (discriminated union) instead of throwing exceptions. This forces consumers to handle the failure case.

- **The Canonical Result Pattern**: Do not invent your own Result type. Use this exact structure for consistency across the project.

  ```typescript
  type Result<T, E> =
    | { success: true; value: T }
    | { success: false; error: E };

  // Example with typed error
  type DivisionError = { code: 'DIV_ZERO'; message: string };

  function divide(a: number, b: number): Result<number, DivisionError> {
    if (b === 0) {
      return { success: false, error: { code: 'DIV_ZERO', message: "Division by zero" } };
    }
    return { success: true, value: a / b };
  }

  // Usage
  const res = divide(10, 2);
  if (res.success) {
    console.log(res.value);
  } else {
    console.error(res.error.message);
  }
  ```

- **Exceptions**: Reserve `throw` for truly exceptional, unrecoverable system errors (e.g., programmer errors, out of memory).
- Always handle Promise rejections.

### Testing

#### Test Coverage Requirements

- **95% Coverage Target**: You MUST aim for 95% test coverage on all code changes. This is a hard requirement, not a suggestion.
- **The 5% Exception**: The remaining 5% is reserved for code that is truly untestable (e.g., platform-specific code paths, certain error handlers that cannot be triggered in tests, or third-party integration edge cases). Use your judgment to determine what qualifies as truly untestable.
- **Coverage Measurement**: You MUST run the test suite with coverage reporting after every code change using `jest --coverage`, `vitest --coverage`, or `c8`/`nyc` for other test runners.
- **Failure Handling**: If coverage is below 95%, you MUST ask the user for help. Do not proceed with incomplete coverage without explicit user guidance.

#### Refactoring for Testability

To achieve 95% coverage, you should proactively refactor and rearchitect the application:

- **Extract Testable Modules**: When encountering difficult-to-reach code paths, extract them into separate, independently testable modules. This makes the code more modular and easier to verify.
- **Dependency Injection via Context Objects**: Use context objects or factory function parameters as the primary mechanism for testability. Pass dependencies rather than importing them directly, allowing tests to substitute mocks or stubs.
- **Adapter Pattern for External Dependencies**: Wrap all external APIs, databases, file systems, and third-party services in adapter modules. This isolates side effects and makes the core logic testable in isolation.
- **Identify and Restructure Untestable Code**: If code cannot achieve coverage, propose architectural changes to the user. Do not accept "untestable" as a final answer without exploring restructuring options.

#### Test Strategy Documentation

- **Mandatory Test Strategy**: For any code changes, you MUST document a test strategy as part of the pre-implementation design review.
- **Strategy Contents**: The test strategy should describe:
  - What will be tested (unit, integration, e2e)
  - How it will be tested (mocks, fixtures, real dependencies)
  - Expected coverage impact
  - Any code that may be excluded from coverage and why

#### Unit & Integration Testing

- Write tests alongside implementation
- Use descriptive test names that explain the behavior
- Test types using utility types like `Expect` or `AssertTrue`
- Mock external dependencies properly
- **Adapter Testing**: Only tests for adapters should invoke actual external dependencies. Other services should use mocks or stubs of the adapters.

#### External API Integration Workflow

1. **Explore the API first** - Use curl or similar to understand actual response shapes before writing any code.
2. **Derive models from real responses** - Application models should be based on actual API responses, not documentation assumptions.
3. **Adapters parse and validate** - Adapters make real calls and deserialize responses into application models. Adapter integration tests should make real API calls whenever practical; use fixtures only when real calls are impractical (rate limits, costs, destructive operations).
4. **Mock adapters internally** - All other services use mock versions of adapters. These mocks return properly-typed application models.
5. **Unit test string manipulation** - URL construction, query parameter handling, ID parsing all need explicit unit tests.

#### End-to-End Testing

- **Functional Requirements Validation**: E2E tests MUST validate that the system aligns with high-level functional requirements.
- **User Clarification**: If the functional requirements are not obvious or clearly documented, you MUST ask the user to clarify them before writing E2E tests.
- **E2E Test Scope**: E2E tests should cover critical user journeys and integration points between services. They complement, but do not replace, unit and integration tests.
- **E2E Frameworks**: Consider using Playwright, Cypress, or Puppeteer for browser-based E2E tests, or supertest for API E2E tests

### Validation

- Runtime validation at system boundaries:

  ```typescript
  import { z } from "zod";

  const UserSchema = z.object({
    id: z.string().uuid(),
    email: z.string().email(),
    age: z.number().min(0).max(120),
  });

  type User = z.infer<typeof UserSchema>;
  ```

## Version Control

- **Conventional Commits**: You MUST use **Conventional Commits** for all git commit messages.
- **User Confirmation**: You MUST always ask the user for confirmation before committing any files.
- **Cleanup**: You MUST clean up any temporary files or build artifacts before committing.

## Checklist Before Committing

- [ ] All TypeScript compilation errors resolved
- [ ] ESLint warnings and errors addressed
- [ ] Tests passing
- [ ] Test coverage meets 95% threshold:
  - Jest: `jest --coverage --coverageThreshold='{"global":{"lines":95}}'`
  - Vitest: `vitest run --coverage --coverage.thresholds.lines=95`
- [ ] No `any` types without justification
- [ ] No `@ts-ignore` comments
- [ ] JSDoc comments for public APIs
- [ ] Types exported where needed
- [ ] No circular dependencies
- [ ] No dead code (unused imports, variables, functions, types)
- [ ] Bundle size impact considered for new dependencies
