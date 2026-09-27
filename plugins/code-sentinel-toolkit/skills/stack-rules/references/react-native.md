# React Native

Covers React Native apps, bare or Expo (managed or prebuild), including apps that live in a
monorepo next to web apps. Load together with `react.md`; this pack wins on conflict.

## Naming
- Platform files `name.ios.tsx` / `name.android.tsx` / `name.native.tsx` / `name.web.tsx`
  export the same interface.
- Screens and navigators named for their route (`OrderDetailScreen`).
- `react.md` naming rules apply.

## Typing
- Navigation params typed (a `ParamList` type per navigator, or typed routes for file-based
  routing); flag `navigation: any` and untyped `route.params`.
- Native module and config values (`Constants.expoConfig?.extra`) validated, not cast.
- `react.md` typing rules apply, including the `{count && ...}` rule, which crashes on
  React Native ("Text strings must be rendered within a `<Text>` component") rather than only
  rendering `0`.

## Error handling
- Network calls handle offline and timeout explicitly; no infinite spinner when the request
  never resolves.
- Errors reported through the app's crash/error reporter with screen context; unhandled
  promise rejections are not silently dropped in release builds.
- Permission requests (camera, location, notifications) handle the denied and
  "denied permanently" results.
- Over-the-air update checks fail safe: a failed update check does not block app start.

## Boundaries
- Secrets never in the JS bundle or app config (anything bundled can be extracted from the
  binary). Tokens in secure storage (Keychain / Keystore via a secure-store library), not
  in plain `AsyncStorage`.
- Screens do not call backend SDKs directly; data comes through hooks or a data layer shared
  with the web app where one exists.
- Shared packages used by both web and native avoid DOM-only or native-only imports unless
  split by platform file.
- N+1: a request per list row rendered in `FlatList` / `FlashList`.

## State and lifecycle
- `AppState` and navigation focus effects (`useFocusEffect`, `navigation.addListener`) remove
  their listeners on cleanup.
- Work that must refresh when the app returns to the foreground or the screen regains focus
  does so (data-fetching library `refetchOnFocus` / `focusManager` wiring, or an explicit
  handler).
- Optimistic updates survive going offline: either queue and retry, or revert with a
  visible error. Flag an optimistic update that stays "succeeded" after a failed request.
- Lists: `keyExtractor` with stable IDs; heavy row components memoized; no inline functions
  that force every row to re-render where the list is long.
- Timers, sockets and subscriptions cleaned up when the screen unmounts, not only when the app
  closes.
- `react.md` state and lifecycle rules apply.

## Testing
- `jest-expo` or the `react-native` Jest preset with React Native Testing Library; query by
  role, label or text.
- Native modules mocked at the module boundary (a `jest.setup` file or `__mocks__/`), not
  inside each test.
- Device-level end-to-end (Detox, Maestro) for critical journeys where the repo has it.
- Coverage minimum read as in `react.md` from the app's own Jest config first.

## PR size exclusions
`ios/Pods/**`, `Podfile.lock`, `ios/**/*.pbxproj` changes produced by tooling, generated
`android/` and `ios/` folders in Expo prebuild projects, `*.snap`, plus `react.md` exclusions.

## Sensitive paths
Auth screens and token storage, deep link and universal link handlers (they accept
untrusted input), push notification handlers, in-app purchase or payment code, app config
that sets permissions or entitlements (`app.json`, `app.config.*`, `Info.plist`,
`AndroidManifest.xml`).
