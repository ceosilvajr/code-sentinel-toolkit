# NestJS

Covers NestJS HTTP, microservice and worker apps on Node.js or Bun, single-app repos and
monorepos with several apps.

## Naming
- Files follow the Nest convention `<name>.<kind>.ts` (`orders.controller.ts`,
  `orders.service.ts`, `create-order.dto.ts`).
- Classes `PascalCase` with the kind suffix (`OrdersService`, `CreateOrderDto`); providers
  injected by type, custom tokens as `UPPER_SNAKE_CASE` constants.
- Universal naming rules apply otherwise (boolean prefixes, verb-led functions). `handle...`
  applies to event/message handlers here, not only UI.

## Typing
- `strict: true` in `tsconfig.json` is the bar; if the repo relaxes it, cite the setting.
- No `any` in DTOs, service signatures or repository return types; `unknown` plus narrowing
  for external payloads.
- DTOs validated at runtime (`class-validator` with a global `ValidationPipe`
  `whitelist: true, forbidNonWhitelisted: true`, or a schema library such as Zod). A type
  annotation alone does not validate a request body.
- Explicit return types on controller and service methods, including `Promise<T>`.
- Truthiness pitfalls as in the universal rules (`if (!amount)`).

## Error handling
- Floating promises: an async call with no `await`, `return` or `.catch()` (enable
  `@typescript-eslint/no-floating-promises` if the repo lints).
- Exceptions caught in a service and turned into a success response; exception filters that
  log and return 200.
- Throw Nest `HttpException` subclasses (or domain errors mapped by a filter) instead of
  returning error objects in a 200 body.
- Logging through the app's logger with request context (request ID, user ID), not
  `console.log`; the error object passed as the stack/trace argument.
- Lifecycle: shutdown hooks (`enableShutdownHooks`, `onModuleDestroy`) close connections and
  consumers.

## Boundaries
- Controller: routing, validation, auth guards, mapping to DTOs. Service: business logic.
  Repository / data module: persistence. Flag data-store or SDK calls in controllers, and
  HTTP types (`Request`, `Response`, `HttpException`) leaking into repositories.
- Auth enforced by guards (`@UseGuards`, global guards), not ad hoc checks inside handlers.
  Flag new routes missing the guard their siblings use.
- Module boundaries: import another module's exported provider, not its internal files.
- Circular dependencies solved with `forwardRef` are a design smell: Suggestion.
- N+1: awaited reads inside `for` / `map` (use batch reads or `Promise.all` with a
  concurrency limit); unbounded scans or `find()` with no pagination.

## State and lifecycle
Not applicable, except request-scoped providers (`Scope.REQUEST`) that hold state across
requests, or singletons that cache per-user data: flag under Boundaries.

## Testing
- Unit tests `*.spec.ts` beside the code; integration or e2e tests in `test/` or `e2e/`
  (`*.e2e-spec.ts`) using `Test.createTestingModule` and a real HTTP call (supertest or fetch).
- Runner: Jest, Vitest or `bun test`. Read the minimum from `coverageThreshold` in
  `jest.config.*` / `package.json`, `test.coverage.thresholds` in `vitest.config.*`, or
  `coverageThreshold` in `bunfig.toml`. In a monorepo, check the app's own config before the
  root one. Cite the file and value.
- Guards, pipes and filters are tested through a request, not only in isolation.

## PR size exclusions
`package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `bun.lock`, `bun.lockb`, generated OpenAPI
specs and clients, `dist/**`.

## Sensitive paths
Guards, auth strategies and middleware, JWT/session handling, role and permission checks,
payment modules, migrations, anything reading or writing personal data, global pipes and
interceptors (they affect every route).
