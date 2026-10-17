# 0001: Stack versions

- **Status:** Accepted
- **Date:** 2026-10-17

## Context

Plan §11.3 requires pinned, mutually compatible dependency versions verified against current documentation. Expo SDK 58 fixes the React Native, React and TypeScript versions for the mobile app. A pnpm monorepo must not contain duplicate React or React Native versions (Expo monorepo guide), so desktop and shared packages follow the versions Expo selects.

## Decision

Exact versions are pinned in `package.json`/`Cargo.toml` and enforced by lockfiles. The table records the selected lines as of this date.

### JavaScript / TypeScript

| Package | Version | Notes |
| --- | --- | --- |
| Node.js | 24.x | Host and CI |
| pnpm | 12.14.0 | Pinned via root `packageManager` field |
| TypeScript | 6.0.3 | Expo SDK 58 pins `~6.0.3`. TypeScript 7.0.2 (latest) is not adopted until Expo supports it |
| React / React DOM | 19.3.0 | Matches Expo SDK 58; one version across the workspace (pnpm `overrides`) |
| Zod | 4.6.x | Runtime schemas in `packages/protocol` |
| Vitest | 5.0.x | Unit tests for shared packages |
| ESLint | 10.13.x + typescript-eslint 8.71.x | Verify typescript-eslint supports TS 6.0 when installed |
| Prettier | 3.9.x | |

### Desktop

| Package | Version | Notes |
| --- | --- | --- |
| Tauri (crate) | 2.12.4 | Locked in smoke build; `tauri-build` 2.x |
| `@tauri-apps/cli` | 2.12.3 | Workspace devDependency, not a global install |
| `@tauri-apps/api` | 2.12.4 | |
| `@tauri-apps/plugin-dialog` | 2.8.x | File selection |
| `@tauri-apps/plugin-fs` | 2.7.x | Scoped filesystem capability (only if needed beyond Rust-side IO) |
| Vite | 8.3.x + `@vitejs/plugin-react` 6.1.x | |
| WebKitGTK | 4.1 (Fedora 43 `webkit2gtk4.1` 2.52.5) | Linux runtime dependency |

### Mobile

| Package | Version | Notes |
| --- | --- | --- |
| Expo SDK | 58 (`expo` 58.0.14) | Development builds only, not Expo Go |
| React Native | 0.88.0 | Fixed by Expo SDK 58 |
| `expo-dev-client` | 58.0.x | |
| `expo-document-picker` | 58.0.x | File selection |
| `expo-file-system` | 58.0.x | Export/save; native module owns transfer bytes |
| Android NDK | 27.1.12297006 | Pinned by React Native 0.88 |
| Android build-tools / platform | 37.0.0 / android-37 | Auto-selected by the Expo template |
| JDK | 21 | `JAVA_HOME=/usr/lib/jvm/java-21-openjdk` |
| Kotlin | 2.2.x | From React Native version catalog |

### Rust (transport candidates, confirmed in ADR 0002)

| Crate | Stable version | Purpose |
| --- | --- | --- |
| tokio | 1.53.x | Async runtime |
| axum / hyper | 0.8.x / 1.12.x | HTTPS listener |
| rustls / tokio-rustls | 0.23.46 / 0.26.6 | TLS 1.3 with custom certificate verification (pinning). `rustls` 0.24 is pre-release, so it is not used |
| rcgen | 0.14.x | Self-signed installation certificate |
| sha2 | 0.11.x | Streaming SHA-256 |
| serde / serde_json | 1.0.x | Wire format |
| uniffi | 0.32.x | Kotlin (and later Swift) bindings if the shared Rust engine is selected |
| keyring | 4.2.x | Linux Secret Service storage for the private key |

## Monorepo rules (from the Expo monorepo guide, SDK 58)

- pnpm isolated `node_modules` are supported since SDK 54. Keep the default isolated linker; fall back to `nodeLinker: hoisted` in `pnpm-workspace.yaml` only if a native module breaks.
- Metro is configured automatically for monorepos (SDK 52+). Do not set `watchFolders`, `extraNodeModules` or `disableHierarchicalLookup` manually.
- Autolinking module resolution is automatic for monorepo apps since SDK 55.
- Exactly one `react` and one `react-native` version in the workspace. Check with `pnpm why --depth=10 react-native`.

## Evidence

- `npm view` / crates.io API queried 2026-10-17.
- Expo SDK 58 / RN 0.88 smoke dev build installed and launched on SM-A366B (see tracker Phase 0).
- Tauri 2.12.4 smoke release build succeeded on Fedora 43.

## Consequences and limitations

- TypeScript is held at 6.0 for the whole workspace until Expo supports 7.x.
- Upgrades are deliberate: bump the version in this ADR (or a superseding ADR) and in the lockfiles together.
