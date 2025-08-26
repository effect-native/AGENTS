# Effect-Native Coding Rules

Guidelines for agents and developers new to Effect in this repository. Follow these rules to write readable, testable, and type-safe code that fits the Effect ecosystem.

## Core Principles

- **Effect-first:** Model all async/IO with `Effect`. Avoid `async/await` in library code; use `Effect.gen` or pipeable combinators.
- **Pipe-first:** Favor the pipeable API (`pipe(…)`) for pure transforms and small chains; use `Effect.gen` when sequencing many steps or mixing awaits and service access.
- **Typed errors:** Model domain errors with `Schema.TaggedError`. Never throw untyped errors from Effect-based APIs; fail with typed errors instead.
- **Dependency injection:** Inject dependencies via `Effect.Service` (preferred) or `Context.Tag`, and provide them with `Layer`.
- **Resource safety:** Manage lifecycles with `Layer.scoped`, `Effect.acquireRelease`, and `Scope`. Don’t leak resources.
- **Edges only run:** Only call `Effect.run*` at program edges (CLIs, servers, scripts, tests). Library code should return `Effect`.

## Effect.gen

- **When to use:** Prefer `Effect.gen` for readable, sequential workflows and service access.
  - Example: `Effect.gen(function* () { const db = yield* Db; const rows = yield* db.query(sql); return rows })`
- **Sequential semantics:** Each `yield*` is sequential. For parallelism, combine effects with `Effect.all`, `Effect.tuple`, or fork fibers.
  - Example: `const [a, b] = yield* Effect.all([fa, fb])`
  - Example: `const fiber = yield* Effect.fork(task); const res = yield* Fiber.join(fiber)`
- **Service access:** `yield* ServiceTag` or `yield* MyService` to access a service from context (see Services below).
- **Error handling:**
  - Convert exceptions: `yield* Effect.try({ try: f, catch: (cause) => new MyError({ cause }) })`
  - Map error types: `program.pipe(Effect.mapError((e) => e instanceof Error ? new MyError({ cause: e }) : e))`
  - Capture result: `const exit = yield* Effect.exit(program)`; assert or branch on `Exit` in tests.
- **Resource safety in gen:** Use `Effect.acquireRelease` inside `Effect.gen`, or prefer `Layer.scoped` for service setup.
- **Anti-patterns:** Don’t mix `await` with `Effect.gen`. Don’t `yield*` non-Effect values. Don’t run effects inside constructors; build effects that construct values instead.

## Services

Prefer `Effect.Service` to declare services with their default `Layer` and (optional) accessors.

```ts
import { Effect, Layer } from "effect"

// 1) Define a service with accessors and dependencies
class Logger extends Effect.Service<Logger>()("Logger", {
  accessors: true,
  effect: Effect.gen(function* () {
    return {
      info: (msg: string) => Effect.sync(() => console.log(msg))
    }
  })
}) {}

// 2) Provide the default layer where you run
const program = Logger.info("Hello").pipe(Effect.provide(Logger.Default))
```

- **Accessors:** With `accessors: true`, methods on the service become functions returning `Effect`, e.g. `Logger.info("msg")`.
- **Dependencies:** Add `dependencies: [OtherService.Default, …]`. Use `Default` to include deps; use `DefaultWithoutDependencies` to get the bare layer.
- **Scoped services:** Use `scoped` when the service acquires resources that must be released (files, sockets, pools). Provide with `Layer.launch` in runtime or `it.scoped` in tests.
- **Use as Tag:** You can `yield* Logger` inside `Effect.gen` to get the concrete implementation if you prefer OO-style usage.
- **When to use Context.Tag:** Use `Context.Tag` for simple DI tokens or when you don’t need a prebuilt default layer/accessors. Otherwise prefer `Effect.Service` for ergonomics.
- **Testing:** Provide stubs via `Layer.succeed(Service, Service.make(stub))` or define a `Test` layer on your service class.

Example (stubbing):

```ts
class Clock extends Effect.Service<Clock>()("Clock", {
  accessors: true,
  sync: () => ({ now: () => Date.now() })
}) {
  static Test = Layer.succeed(this, new Clock({ now: () => 123 }))
}

// In test: Effect.provide(Clock.Test)
```

## Schema.TaggedError

Use `Schema.TaggedError` to define typed, serializable error classes with consistent `_tag` and schema.

```ts
import { Schema, Effect } from "effect"

class UserNotFound extends Schema.TaggedError<UserNotFound>()(
  "UserNotFound",
  { id: Schema.String }
) {}

class Unauthorized extends Schema.TaggedError<Unauthorized>()(
  "Unauthorized",
  {}
) {}

// Failing with a tagged error
const findUser = (id: string) => Effect.fail(new UserNotFound({ id }))
```

- **Message:** You may define a `get message()` in the class for custom rendering, or include a `message: Schema.String` field.
- **Cause:** Use `cause: Schema.Defect` field if you want to carry an underlying throwable (`Error`, unknown failure) for diagnostics.
- **Composable unions:** Use union types of TaggedErrors for function error channels, e.g. `Effect<A, UserNotFound | Unauthorized, R>`.
- **At boundaries:** Map third-party errors into your domain errors at module edges. Inside your domain, raise your own tagged errors only.
- **HTTP APIs:** When using `@effect/platform` Http APIs, add error schemas via `.addError(Unauthorized, { status: 401 })`, etc.

## Testing and TDD (Red/Green/Refactor)

Use `@effect-native/bun-test` or `@effect/vitest` to write tests as Effects without `runPromise`.

- **Red:** Start with a failing `it.effect` test that returns an `Effect`.
  - Example:
    ```ts
    import { it, expect } from "@effect-native/bun-test"
    import { Effect } from "effect"
    it.effect("divides numbers", () =>
      Effect.gen(function* () {
        const result = yield* divide(4, 2)
        expect(result).toBe(2)
      })
    )
    ```
- **Green:** Implement with `Effect.gen`, inject dependencies via `Layer`, and fail with `Schema.TaggedError` where needed.
- **Refactor:** Extract dependencies into services, enable `accessors`, compose layers, keep tests fast/deterministic with stubs and `TestClock`.

Key testing practices:

- **it.effect:** Default mode; injects `TestContext` (e.g. `TestClock`). Place assertions inside the returned `Effect`.
- **it.scoped:** Use when the program needs a `Scope` (e.g. `Effect.acquireRelease`, `Layer.scoped`).
- **it.live:** Run with live environment for integration-style checks (e.g. real time, console). Use sparingly.
- **Failures:** Capture exits with `Effect.exit` or use `it.effect.fails` for temporarily expected failures.
- **Time control:** Use `TestClock.adjust("1 second")` to advance simulated time deterministically.
- **Stubs:** Provide test layers: `Effect.provide([Service1.Test, Service2.Default])` or `Layer.succeed(Service, Service.make(stub))`.

Example (service + test):

```ts
// service
class UserRepo extends Effect.Service<UserRepo>()("UserRepo", {
  accessors: true,
  effect: Effect.succeed({
    findById: (id: string) => Effect.fail(new UserNotFound({ id }))
  })
}) {}

// test
import { it, expect } from "@effect-native/bun-test"
import { Effect, Exit } from "effect"

it.effect("findById returns not found", () =>
  Effect.gen(function* () {
    const exit = yield* Effect.exit(UserRepo.findById("1").pipe(Effect.provide(UserRepo.Default)))
    expect(exit).toStrictEqual(Exit.fail(new UserNotFound({ id: "1" })))
  })
)
```

## Practical Do/Don’t

- **Do:**
  - Use `Effect.gen` for readable sequencing and service access.
  - Use `Effect.Service` with `accessors: true` and provide via `Service.Default` in callers/tests.
  - Model all domain errors with `Schema.TaggedError` and document them.
  - Compose `Layer`s in app wiring; keep library code context-polymorphic.
  - Keep tests hermetic with stubs, `TestClock`, and `it.effect`.

- **Don’t:**
  - Don’t call `Effect.run*` in libraries; only at app edges and tests.
  - Don’t throw untyped errors; map them into tagged errors.
  - Don’t use global singletons; use services + layers.
  - Don’t mix `async/await` with Effect. Use Effect operators throughout.

## Cheatsheet

- **Access a service:** `const svc = yield* MyService`
- **Use accessor:** `yield* MyService.doThing(arg)` (with `accessors: true`)
- **Provide deps:** `Effect.provide([A.Default, B.Default])`
- **Parallel:** `yield* Effect.all([fa, fb, fc])`
- **Exit:** `const exit = yield* Effect.exit(eff)`
- **Tagged error:**
  ```ts
  class E extends Schema.TaggedError<E>()("E", { msg: Schema.String }) {}
  yield* Effect.fail(new E({ msg: "bad" }))
  ```
- **Test:** `it.effect("name", () => Effect.gen(function* () { /* assertions */ }))`
