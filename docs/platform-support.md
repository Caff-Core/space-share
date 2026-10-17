# Platform Support

Status per target. Update with evidence at each phase (see `IMPLEMENTATION_PHASES.md`).

| Target | Track | Build status | Physical-device evidence | Permissions | Known limitations |
| --- | --- | --- | --- | --- | --- |
| Linux desktop (Fedora 43 host) | Current | Not built | None | TBD | TBD |
| Linux desktop (Debian/Ubuntu) | Current | Not built | None | TBD | `.deb` validated in container/VM only |
| Android | Current | Not built | None | TBD | Foreground transfers only (MVP) |
| Windows | Deferred | — | — | — | Excluded for now |
| macOS | Deferred | — | — | — | No macOS host |
| iOS | Deferred | — | — | — | No macOS/Xcode host; binding path unproven |

## Device pairs

| Pair | Status |
| --- | --- |
| Linux ↔ Android | Not tested |
| Linux ↔ Linux | Not tested |
| Android ↔ Android | Not tested |
