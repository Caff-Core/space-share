# Platform Support

Status per target. Update with evidence at each phase (see `IMPLEMENTATION_PHASES.md`).

| Target | Track | Build status | Physical-device evidence | Permissions | Known limitations |
| --- | --- | --- | --- | --- | --- |
| Linux desktop (Fedora 43 host) | Current | Smoke Tauri 2.12.4 build only | None | Firewall zone `FedoraWorkstation` allows TCP 1025–65535; Secret Service for keys | WebKitGTK 4.1 runtime dependency |
| Linux desktop (Debian/Ubuntu) | Current | Not built | None | TBD | `.deb` validated in container/VM only |
| Android (SM-A366B, arm64) | Current | Smoke Expo SDK 58 dev build installed | None | Planned: `INTERNET`, `ACCESS_NETWORK_STATE`, `ACCESS_WIFI_STATE`; SAF for files | Foreground transfers only (MVP); adb drops intermittently on test phone |
| Windows | Deferred | — | — | — | Excluded for now |
| macOS | Deferred | — | — | — | No macOS host |
| iOS | Deferred | — | — | — | No macOS/Xcode host; binding path unproven |

## Device pairs

| Pair | Status |
| --- | --- |
| Linux ↔ Android | Not tested |
| Linux ↔ Linux | Not tested |
| Android ↔ Android | Not tested |

## Toolchain (host)

See [ADR 0001](decisions/0001-stack-versions.md) for pinned versions. Host: Fedora 43, Node 24, pnpm 12.14, Rust 1.97, JDK 21, Android NDK 27.1.12297006.

## Network requirements

- Both peers on the same LAN segment with client isolation disabled (guest Wi-Fi networks usually block peer traffic).
- Host firewall must allow the receive listener's TCP port. The app never changes firewall settings itself.
