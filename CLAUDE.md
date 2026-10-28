# CLAUDE.md

Guidance for Claude Code when working in this NestJS project. Read this before making changes.

## Project Overview

- **Framework:** NestJS (TypeScript, strict mode)
- **Runtime:** Node.js (LTS)
- **Testing:** Jest (unit) + Supertest (e2e)
- **Package manager:** npm (adjust if the project uses pnpm/yarn)

## Common Commands

```bash
npm run start:dev      # Run the app in watch mode
npm run build          # Compile TypeScript
npm run lint           # ESLint
npm run format         # Prettier
npm run test           # Unit tests
npm run test:e2e       # End-to-end tests
npm run test:cov       # Coverage report
```

Before declaring a task complete, run `npm run lint`, `npm run build`, and `npm run test` and make sure all pass.

---

## How to Work With Me

### Push back when something seems wrong

You are a collaborator, not just an executor. **Do not silently implement an approach you believe is flawed.**

- If a requested approach is wrong, risky, or there is a clearly better alternative, **say so before writing code**, explain why, and propose the alternative.
- If there are meaningful **trade-offs** (performance vs. readability, consistency vs. speed of delivery, new dependency vs. hand-rolled code, breaking API changes, migration risk, etc.), lay them out briefly and give a recommendation.
- If requirements are ambiguous in a way that changes the design, ask a focused question instead of guessing.
- If you notice an existing bug, security issue, or design smell adjacent to the task, flag it — but don't fix it outside the task's scope without asking.
- Disagreement is welcome. Be direct and specific; "this will cause N+1 queries because X" beats "this might not be ideal."

Once a decision has been made and acknowledged, implement it without re-litigating.

---

## Code Style & Clean Code

### General principles

- **Readable over clever.** Code is read far more than it is written.
- **Small, focused functions** that do one thing. If a function needs a comment explaining each section, split it.
- **Meaningful names.** `findActiveUsersByOrg` over `getData`. Booleans read as questions: `isActive`, `hasAccess`.
- **No magic numbers or strings** — extract to named constants or enums.
- **Early returns** over deeply nested `if`/`else`.
- **DRY, but not prematurely.** Duplicate twice, abstract on the third time.
- **No dead code**, commented-out code, or unused imports.
- **Comments explain _why_, not _what_.** The code should explain what.
- Keep files reasonably small; one class/primary export per file.

### TypeScript

- `strict: true` — no implicit `any`.
- **Never use `any`.** Use `unknown` and narrow, or define a proper type.
- Prefer `type`/`interface` for data shapes; prefer union types and discriminated unions over boolean flags.
- Use `readonly` on properties and `ReadonlyArray<T>` / `readonly T[]` for inputs that must not be mutated.
- Avoid non-null assertions (`!`). Handle the `undefined` case explicitly.
- Explicit return types on public/exported functions and service methods.

---

## Functional Programming Style

Take inspiration from functional programming. The goal is predictable, testable code with minimal hidden state.

### Immutability — `const` everywhere, never `let`

- **Always use `const`.** Do **not** use `let`. Do **not** use `var`.
- If you feel you need `let`, it is a signal to restructure: use a ternary, an early return, a helper function, or `reduce`/`map`/`filter`.
- Never mutate function arguments, DTOs, or shared objects. Return new values instead.
- Prefer spread / `structuredClone` / `Object.freeze` for creating updated copies.
- Prefer non-mutating array methods: `map`, `filter`, `reduce`, `flatMap`, `toSorted`, `toReversed`, `toSpliced`, `with`, `concat`. Avoid `push`, `pop`, `splice`, `sort`, `reverse` on shared data.

```ts
// ❌ Avoid
let total = 0;
for (const item of items) {
  total += item.price * item.quantity;
}

let status;
if (user.isVerified) {
  status = 'active';
} else {
  status = 'pending';
}

// ✅ Prefer
const total = items.reduce((sum, item) => sum + item.price * item.quantity, 0);

const status = user.isVerified ? 'active' : 'pending';
```

```ts
// ❌ Avoid — mutates the input
function applyDiscount(order: Order): Order {
  order.total = order.total * 0.9;
  return order;
}

// ✅ Prefer — returns a new value
const applyDiscount = (order: Readonly<Order>): Order => ({
  ...order,
  total: order.total * 0.9,
});
```

### Pure functions & side effects

- Keep **business logic in pure functions**: same input → same output, no I/O, no hidden state.
- Push side effects (DB, HTTP, queues, logging, time, randomness) to the **edges** — controllers, repositories, and thin service orchestration methods.
- Inject non-determinism (clock, UUID generator) so it can be controlled in tests.
- Compose small functions rather than writing large procedural methods.
- Prefer declarative pipelines (`map`/`filter`/`reduce`) over imperative loops with accumulators.

### Errors

- Don't use exceptions for normal control flow inside domain logic. Model expected failures explicitly (e.g. a `Result`/`Either`-style return or a discriminated union).
- Translate domain failures into NestJS `HttpException`s at the controller/service boundary (or via an exception filter).

---

## NestJS Conventions

### Structure

- **Feature modules:** one module per bounded feature (`users/`, `orders/`, ...). Each contains its `*.module.ts`, `*.controller.ts`, `*.service.ts`, DTOs, entities, and tests.
- **Controllers are thin:** parse/validate input, call a service, map the result to a response. No business logic.
- **Services orchestrate;** pure helper functions hold the actual logic.
- **Repositories / data access** are isolated behind providers so they can be mocked.

```
src/
  users/
    dto/
      create-user.dto.ts
    users.controller.ts
    users.controller.spec.ts
    users.service.ts
    users.service.spec.ts
    users.logic.ts          # pure functions
    users.logic.spec.ts
    users.module.ts
```

### Dependency injection

- Use constructor injection with `private readonly` dependencies.
- Depend on abstractions (interfaces + injection tokens) where swapping implementations is likely.
- No `new`-ing up services inside other services.

### Validation & DTOs

- Validate all incoming data with `class-validator` / `class-transformer` (or Zod) and a global `ValidationPipe` with `whitelist: true` and `forbidNonWhitelisted: true`.
- DTO properties should be `readonly`.
- Never return raw database entities from controllers — map to response DTOs.

### Configuration & secrets

- Use `@nestjs/config`; read config through a typed config service, not `process.env` scattered in code.
- Never hard-code or commit secrets.

### Async

- Use `async`/`await`; never leave promises floating.
- Run independent async work concurrently with `Promise.all`.

---

## Testing — Required for Every Change

**All new features and behaviour changes must include tests.** A task is not done until tests are written and passing.

- **New feature →** unit tests for the logic and service, plus an e2e test for any new/changed endpoint.
- **Bug fix →** first write a test that reproduces the bug (fails), then fix it (passes).
- **Refactor →** existing tests must still pass; add tests if coverage was missing.
- Co-locate unit tests next to the source as `*.spec.ts`; e2e tests live in `test/` as `*.e2e-spec.ts`.
- Pure functions are tested directly without mocks. Services are tested with `Test.createTestingModule` and mocked providers.
- Cover the happy path, edge cases (empty, null/undefined, boundaries), and failure paths.
- Tests follow **Arrange / Act / Assert** and have descriptive names: `it('returns 404 when the user does not exist')`.
- Tests are deterministic and independent — no shared mutable state, no reliance on order, no real network calls. Fake the clock and ID generators.
- The same `const`-only, immutable style applies to test code.
- Don't weaken or delete an existing test to make it pass without explaining why.

```ts
describe('applyDiscount', () => {
  it('returns a new order with 10% off the total', () => {
    const order = { id: '1', total: 100 } as const;

    const result = applyDiscount(order);

    expect(result.total).toBe(90);
    expect(order.total).toBe(100); // input not mutated
  });
});
```

---

## Definition of Done

- [ ] Approach agreed (concerns and trade-offs raised up front)
- [ ] Code follows the clean code + functional guidelines above (no `let`, no mutation, no `any`)
- [ ] Tests added/updated for every new feature or change
- [ ] `npm run lint`, `npm run build`, and `npm run test` all pass
- [ ] No unrelated changes slipped in
- [ ] Summary of what changed, and any known limitations or follow-ups
