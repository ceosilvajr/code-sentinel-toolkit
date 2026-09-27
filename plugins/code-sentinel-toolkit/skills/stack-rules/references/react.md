# React

Covers React components, hooks and shared UI packages. Next.js and React Native apps load
this pack plus their own; their pack wins on conflict.

## Naming
- Components `PascalCase`, one exported component per file where the repo does so.
- Hooks start with `use`; a function that calls hooks must be a hook or a component.
- Event handler functions `handle...`, the props that receive them `on...`.
- Boolean props and state with `is`/`has`/`should`/`can`.

## Typing
- Props typed explicitly (`type Props` / `interface Props`); no `any` in props, context values
  or reducer actions.
- Reducer actions and component variants as discriminated unions, not optional-everything
  props.
- `children` typed (`React.ReactNode`) rather than `any`.
- Truthiness in JSX: `{count && <Badge />}` renders `0`. Require `count > 0 &&` or a ternary.

## Error handling
- Async work in effects or handlers with no error branch: the user sees nothing when it fails.
- Error boundaries around subtrees that can throw during render, where the repo uses them.
- Errors reported through the app's reporter (for example an error-tracking SDK), not only
  `console.error`.

## Boundaries
- Components do not call data stores or backend SDKs directly; data comes through a hook,
  a data-fetching library or props.
- Business rules live in plain functions or hooks, not inside JSX.
- Shared UI packages do not import app-specific code.
- N+1: a request per list item (a fetch inside a mapped child component) where one batched
  request would do.

## State and lifecycle
- Effects: correct dependency arrays (follow `react-hooks/exhaustive-deps`), cleanup for
  subscriptions, timers, listeners and `AbortController` on unmount.
- Derived values computed during render, not copied into state by an effect.
- Optimistic updates: revert and show an error when the request fails. With React Query /
  SWR / RTK Query, use their `onError` / rollback context.
- Cache invalidation after mutations: invalidate or update every query key the mutation
  makes stale.
- Keys: stable IDs, not array indexes, for lists that reorder or change.
- Race conditions: a slower earlier request must not overwrite a newer one (ignore stale
  responses or abort).

## Testing
- Testing Library (`@testing-library/react`): query by role and label, assert what the user
  sees; no assertions on internal state or implementation details.
- Runner: Jest or Vitest. Read the minimum from `coverageThreshold` in `jest.config.*` /
  `package.json`, or `test.coverage.thresholds` in `vitest.config.*`. In a monorepo, check
  the package's own config before the root one. Cite the file and value.
- Network mocked at the boundary (MSW or a fetch mock), not by mocking the component's own
  hooks.
- Loading, empty and error states each have a test for data-showing components.

## PR size exclusions
Lockfiles (`package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `bun.lock`, `bun.lockb`),
`*.snap` snapshot files, generated icon or asset index files, Storybook static builds.

## Sensitive paths
Auth screens and session handling, token storage, payment forms, components rendering
user-supplied HTML (`dangerouslySetInnerHTML`), feature-flag or permission gating in the UI.
