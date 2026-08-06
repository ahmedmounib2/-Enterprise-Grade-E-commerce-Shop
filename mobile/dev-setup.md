# Mobile Dev Setup

Expo Go is not a supported way to run this app. Use the
[custom development client](./custom-dev-setup.md) instead.

## Why Expo Go doesn't work here

Every network request the app makes goes through `packages/api-client`, whose React Native build
unconditionally installs certificate pinning via `react-native-ssl-public-key-pinning`
(`packages/api-client/sslPinningAdapter.native.js`, selected by Metro's platform-specific module
resolution for every React Native target — this is a bundler-time rule, not a runtime check, so it
applies identically whether the bundle later runs in Expo Go or a custom dev client).
`react-native-ssl-public-key-pinning` is a third-party native module outside Expo Go's bundled
module set, so Expo Go cannot load it. Because `AuthProvider` (`mobile/src/auth/AuthProvider.js`)
calls `setBaseURL()` from the app root at launch, this is not a narrow edge case — it fails on the
app's first network call, before any meaningful screen renders.

There is no code fix that preserves Expo Go support without weakening security. The alternative —
skipping pinning specifically under Expo Go — is exactly the kind of runtime downgrade path the
pinning implementation is deliberately built to prevent (see `mobile/docs/ssl-pinning.md` and the
Certificate Pinning section of the Architecture page: pinning fails closed, with no way to disable
it at runtime). Expo Go support is not planned.

## What to use instead

[`mobile/custom-dev-setup.md`](./custom-dev-setup.md) covers the custom development client: Android
toolchain setup, device connectivity, environment profiles, and the daily development loop. It is
the only supported way to run and test this app locally.
