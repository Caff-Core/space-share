# Space Share: Implementation Phases Tracker

Tracks execution of [`FILE_SHARING_AGENT_PLAN.md`](./FILE_SHARING_AGENT_PLAN.md) (the spec) from setup to deployment. The spec holds the rationale; this file holds the work. References like "plan §6" point to spec sections.

**Last updated:** 2026-10-17

## Rules for this file

- **Done** means `[x]` plus evidence: a command output summary, commit hash, CI run link, or real-device test note in the phase's evidence log. Mocks alone never complete a task or gate (plan §9).
- A phase is complete only when every exit-gate item is checked. Partial phases stay `In progress`.
- Do not start a phase before the previous exit gate passes, except Phase 0 tasks that are pure setup.
- Record blockers in [Known blockers](#known-blockers) rather than silently skipping a task.
- Keep the repository buildable at the end of every phase.

## Scope for this track

| Item | Decision |
| --- | --- |
| First validation pair | Linux desktop (Fedora host) + physical Android phone |
| Mobile–mobile proof | Android ↔ Android (iOS half of plan §4.5 is an open blocker) |
| Desktop packages | AppImage, `.deb`, `.rpm` (Tauri bundler); Flatpak as stretch |
| Mobile package | Sideloadable signed APK (self-generated release key) |
| Deployment | GitHub Releases; no app stores, no production signing |
| CI | GitHub Actions, Linux runners |
| Deferred | Windows, macOS, iOS (see [Deferred platforms](#phase-10-deferred-platforms)) |

## Status summary

| Phase | Name | Status |
| --- | --- | --- |
| 0 | Environment and repository setup | In progress |
| 1 | M0: Inspect and document | Not started |
| 2 | M1: Feasibility spike | Not started |
| 3 | M2: Monorepo foundation | Not started |
| 4 | M3: One secure file | Not started |
| 5 | M4: Batches and failures | Not started |
| 6 | M5: Usable MVP | Not started |
| 7 | M6: Hardening and validation | Not started |
| 8 | M7: Packaging and release | Not started |
| 9 | M8: Post-MVP discovery | Not started |
| 10 | Deferred platforms | Blocked |

---

## Phase 0: Environment and repository setup

Goal: a host that can build the Tauri desktop app and an Expo Android development build, plus a version-controlled repo with CI.

### Host toolchain

- [x] Node.js installed (v24.15.0)
- [x] pnpm installed (12.14.0, `~/.local/share/pnpm/bin`; ensure it is on `PATH` for non-login shells and CI)
- [x] Rust toolchain installed (rustc 1.97.0, cargo 1.97.0)
- [x] Java JDK installed (OpenJDK 25.0.4.1). Verify compatibility with the Gradle/AGP version Expo pins; install JDK 17 if required.
- [x] Android SDK present (platforms 34, 35, 36.1; build-tools; cmdline-tools; emulator)
- [x] Android NDK present (30.0.16138531). Verify against the NDK version required by Expo/React Native.
- [x] `adb` installed (1.0.41)
- [ ] `ANDROID_HOME` / `ANDROID_SDK_ROOT` and `JAVA_HOME` exported in shell profile
- [ ] Tauri Linux prerequisites installed: `webkit2gtk4.1-devel`, `libappindicator-gtk3-devel`, `librsvg2-devel`, `openssl-devel` (present), plus `curl wget file` and the C toolchain group (see Tauri prerequisites docs)
- [ ] `rpm-build` installed for `.rpm` bundling; `dpkg`/`fakeroot` available for `.deb` if Tauri needs them on Fedora
- [ ] Rust components: `rustfmt`, `clippy`; Android Rust targets only if the mobile native module uses Rust (decided in Phase 2)
- [ ] `cargo install tauri-cli` (or use `@tauri-apps/cli` via pnpm) at a pinned version

### Devices

- [ ] Physical Android phone in developer mode, USB debugging enabled, visible in `adb devices`
- [ ] Second Android device (physical preferred) available for Android ↔ Android tests
- [ ] Phone and host on the same Wi-Fi with no client isolation; host firewall rule documented for the listener port

### Repository bootstrap

- [ ] `git init`, `.gitignore` (node_modules, target, build outputs, `.env*`, keystores)
- [ ] Remote repository created on GitHub
- [ ] Root `README.md` with project summary and link to the spec and this tracker
- [ ] `docs/platform-support.md` stub (plan §11.7)
- [ ] `docs/decisions/` folder with ADR template (`0000-template.md`)
- [ ] CI skeleton workflow (checkout, pnpm install, placeholder job) running green on push

### Exit gate

- [ ] `pnpm`, `cargo`, `adb` and `java` all work from a non-login shell
- [ ] A throwaway `create-tauri-app` builds and opens a window on the host (then deleted)
- [ ] A throwaway Expo development build installs and runs on the physical phone (then deleted)
- [ ] First commit pushed and CI skeleton passes

### Evidence log

- 2026-10-17: Toolchain inventory taken on Fedora 43 (Linux 7.2.6). Node 24.15.0, pnpm 12.14.0, Rust 1.97.0, OpenJDK 25.0.4.1, Android SDK + NDK 30.0.16138531, adb 1.0.41. Not installed: webkit2gtk4.1-devel, libappindicator-gtk3-devel, librsvg2-devel. No device attached to adb. Repo not yet a git repository.

---

## Phase 1: M0: Inspect and document

Goal: pinned versions, documented defaults and a protocol outline before any code (plan §9 M0).

- [ ] Pin stable versions: Tauri 2.x, React, Vite, TypeScript, Expo SDK, React Native, Zod; record in ADR `0001-stack-versions.md`
- [ ] Check Expo SDK monorepo + pnpm guidance (node-linker / hoisting requirements)
- [ ] `docs/architecture.md`: process boundaries, native-owns-bytes rule, adapter command/event list (plan §5)
- [ ] `docs/protocol-v1.md` outline: operations table, manifest fields, decimal-string sizes, version negotiation, limits (plan §6)
- [ ] Central limits table drafted (manifest size, file count, filename length, rate, unauthenticated lifetime) (plan §7)
- [ ] `docs/platform-support.md` populated with the target matrix for this track (Linux, Android) and deferred targets

### Exit gate

- [ ] Versions pinned and recorded
- [ ] Architecture and protocol outline reviewed and committed

### Evidence log

_None yet._

---

## Phase 2: M1: Feasibility spike

Goal: prove the risky native parts on Linux ↔ Android before building UI (plan §4). Spike code may live in `spikes/` and be discarded.

- [ ] **4.1** File selection, readable handles, streaming to disk and export/save on Linux (Tauri) and Android (Expo dev build)
- [ ] **4.2** Native foreground HTTPS listener on Android; outbound client on both; transfer works with **each** side as receiver
- [ ] **4.3** Authenticated TLS with certificate/public-key pinning on desktop (Rust) and Android, validated in the actual transport library (not JS `fetch`)
- [ ] **4.4** Transfer a file larger than memory buffers (≥1 GiB if storage permits); record baseline/peak/sustained memory; cancel midway and confirm cleanup
- [ ] **4.5a** Android ↔ Android transfer with the chosen native transport
- [ ] **4.5b** iOS binding path: **blocked** (no macOS host). Record as open blocker; keep the native module interface platform-neutral.
- [ ] **4.6** ADR `0002-native-transport-and-security.md`: libraries, versions, licenses, maintenance status, permissions, limitations, how to run the proof
- [ ] Decide whether the Android native module is Kotlin-only or Rust shared with desktop (via JNI/uniffi); record in ADR

### Exit gate

- [ ] Both directions work on Linux ↔ Android with binary streaming, pinned peer identity and SHA-256 verification
- [ ] Memory does not grow with file length
- [ ] ADR committed with runnable proof instructions

### Evidence log

_None yet._

---

## Phase 3: M2: Monorepo foundation

Goal: reproducible workspace with shared contracts and app shells (plan §5, §9 M2).

- [ ] `pnpm-workspace.yaml` with `apps/*` and `packages/*`; root scripts (`lint`, `typecheck`, `test`, `build`)
- [ ] Strict shared `tsconfig` base; ESLint + Prettier configured
- [ ] `packages/protocol`: Zod schemas, derived types, protocol version constant, JSON fixtures
- [ ] `packages/transfer-core`: pure per-file/batch state machine and byte-weighted progress (no React/platform imports)
- [ ] `packages/design-tokens`: colours, spacing, typography
- [ ] `packages/test-fixtures`: valid and malformed interoperability cases
- [ ] `apps/desktop`: Tauri 2 + React + Vite shell with restrictive CSP and minimal capabilities
- [ ] `apps/mobile`: Expo dev-build shell resolving workspace packages (Metro monorepo config)
- [ ] Rust protocol types in `src-tauri` checked against shared fixtures (drift test)
- [ ] Android native types checked against the same fixtures
- [ ] CI: typecheck, lint, unit tests, `cargo fmt --check`, `cargo clippy -D warnings`, `cargo test`
- [ ] Lockfiles committed; fresh clone → install → build verified

### Exit gate

- [ ] Fresh clone builds desktop app and Android dev build with no manual steps beyond documented setup
- [ ] CI green on all checks

### Evidence log

_None yet._

---

## Phase 4: M3: One secure file

Goal: one file, end to end, secure, both directions on real devices (plan §6, §9 M3).

- [ ] Installation key pair generated; private key in platform-protected storage (Linux secret service/keyring; Android Keystore)
- [ ] Editable display name and stable local identity
- [ ] Receive session: visible receiving state, start/stop, expiring single-use pairing secret
- [ ] QR payload (address, port, version, fingerprint, secret) generated on receiver, scanned on sender
- [ ] Manual connection: endpoint + fingerprint verification + pairing secret
- [ ] Pair/session operation issuing scoped, expiring credentials (established implementation, no custom crypto)
- [ ] Create offer / read offer status / accept / reject
- [ ] Upload file as native stream into app-owned temp file
- [ ] Commit file: byte count + SHA-256 match, atomic move to final location, acknowledgement
- [ ] Filename sanitization and collision-safe naming (plan §7)
- [ ] Secrets redacted from logs
- [ ] Demonstrated Linux → Android and Android → Linux on real devices

### Exit gate

- [ ] Both directions verified on physical devices with evidence recorded
- [ ] Wrong fingerprint and reused pairing secret rejected (manual test minimum; automated in Phase 7)

### Evidence log

_None yet._

---

## Phase 5: M4: Batches and failures

Goal: correct batch behavior under failure (plan §7, §9 M4).

- [ ] Sequential queue: one active stream per batch; global receive limits enforced
- [ ] Byte-weighted aggregate progress; zero-byte files complete via acknowledgement
- [ ] Throttled native progress events; speed calculation
- [ ] Cancel offer: closes streams, deletes unfinished temp files, keeps completed files
- [ ] Batch outcomes: completed, partially completed, rejected, canceled, failed
- [ ] Per-file retry restarts failed file from zero; completed files never resent
- [ ] Retry idempotency: duplicate retries cannot duplicate committed files
- [ ] Reconnect reconciliation: lost acknowledgement does not imply failure; interface snapshot resync
- [ ] Source-change detection (metadata recheck or stable handle)
- [ ] Storage estimate check before acceptance; write-failure handling during transfer
- [ ] Stale partial cleanup at startup with bounded retention

### Exit gate

- [ ] Partial-failure batch shows completed files and retries only the failed ones on real devices
- [ ] Duplicate retry test passes

### Evidence log

_None yet._

---

## Phase 6: M5: Usable MVP

Goal: complete MVP feature set from plan §2 with usable UI (plan §8, §9 M5).

- [ ] Views: nearby/connect, outgoing selection, incoming approval, active/history
- [ ] Empty, permission-denied, disconnected and partial-failure states
- [ ] Desktop: multi-file picker and drag-and-drop
- [ ] Android: multi-file picker and explicit open/export action for received files
- [ ] Incoming approval shows sender identity, file names, sizes and total
- [ ] Local history (metadata only), configurable retention, clear-history action
- [ ] Never auto-open received files
- [ ] Accessibility: labels, desktop keyboard navigation, screen-reader-friendly progress
- [ ] Android: local-network/storage permission prompts and foreground-requirement messaging
- [ ] Actionable errors for firewall, guest network/client isolation and denied permissions
- [ ] Tauri capabilities narrowed to selected sources and configured destinations

### Exit gate

- [ ] Every MVP bullet in plan §2 demonstrable on Linux ↔ Android
- [ ] Permissions and storage behavior documented in `platform-support.md`

### Evidence log

_None yet._

---

## Phase 7: M6: Hardening and validation

Goal: automated and real-device evidence for every consequential behavior (plan §10).

### Automated tests

- [ ] Schema validation, malformed manifests, incompatible versions, shared Rust/Kotlin/TS fixtures
- [ ] State transitions, partial batches, retry idempotency, zero-byte files, byte-weighted progress
- [ ] Wrong fingerprints, expired/replayed pairing secrets, unauthorized uploads, offer/file credential isolation
- [ ] Integrity mismatch, truncated streams, disk write failure, path traversal, filename collisions
- [ ] Cancellation cleanup, lost acknowledgement / reconnect reconciliation

### Real-device matrix (both directions each)

- [ ] Linux ↔ Android
- [ ] Linux ↔ Linux (second machine or VM with bridged network)
- [ ] Android ↔ Android

### Real-device scenarios

- [ ] Binary files, Unicode and long filenames, duplicate names, empty files
- [ ] Many-file batch (≥100 files)
- [ ] ≥1 GiB file with memory baseline/peak/sustained recorded
- [ ] Wi-Fi loss, receiver exit, source deletion/change, permission denial, low storage
- [ ] Android background/foreground transitions during transfer
- [ ] Internet disconnected, LAN active: full transfer succeeds

### Exit gate

- [ ] All automated tests green in CI
- [ ] Device matrix and scenarios recorded with evidence; failures logged as blockers, not hidden
- [ ] Offline end-to-end demonstration recorded

### Evidence log

_None yet._

---

## Phase 8: M7: Packaging and release

Goal: installable artifacts published on GitHub Releases.

### Linux desktop

- [ ] App metadata: identifier, icons, `.desktop` entry, version from single source
- [ ] AppImage built and launched on Fedora
- [ ] `.rpm` built; install/uninstall smoke test on Fedora
- [ ] `.deb` built; install/uninstall smoke test on Ubuntu/Debian (container or VM)
- [ ] Each package: transfer smoke test with Android after install
- [ ] Runtime dependencies declared correctly per package format (webkit2gtk etc.)
- [ ] _Stretch:_ Flatpak manifest; file access via portals verified; local network listener works inside sandbox

### Android

- [ ] Release keystore generated and stored outside the repo; backup location documented
- [ ] Release APK built (local Gradle / EAS local build) with production-style config
- [ ] APK sideload-installed on physical phone; transfer smoke test with Linux build
- [ ] Permissions in manifest reviewed and minimal

### Release pipeline

- [ ] CI release workflow on tag: builds AppImage, `.deb`, `.rpm` and APK, attaches to GitHub Release
- [ ] Signing secrets stored as CI secrets; never committed
- [ ] SHA-256 checksums file published with artifacts
- [ ] `CHANGELOG.md` and release notes with known limitations and supported platforms
- [ ] `platform-support.md` final for this release
- [ ] Tag `v0.1.0` and publish GitHub Release

### Exit gate

- [ ] Fresh download of each artifact installs and completes a Linux ↔ Android transfer offline

### Evidence log

_None yet._

---

## Phase 9: M8: Post-MVP discovery

Goal: automatic nearby discovery without weakening trust (plan §8, §9 M7). Start only after Phase 8.

- [ ] mDNS service publish/browse on Linux and Android with minimal metadata
- [ ] Discovery never treated as authentication; fingerprint pinning still required
- [ ] QR/manual path retained and tested
- [ ] Android multicast permissions and fallback behavior retested
- [ ] Next subsequent-release features prioritized (plan §2): parallel transfers, resume, folders, share-sheet, trusted devices, background, multi-recipient

### Evidence log

_None yet._

---

## Phase 10: Deferred platforms

Excluded from this track. Architecture must not preclude them.

| Platform | Blocked by | Needed to unblock |
| --- | --- | --- |
| Windows | Excluded by decision; no Windows host | Windows machine or CI runner; MSI/NSIS bundling; firewall prompt handling |
| macOS | No macOS host | Mac or macOS CI runner; Apple developer signing/notarization for distribution |
| iOS | No macOS/Xcode; plan §4.5 iOS binding unproven | Mac + Xcode; iOS native listener module; local network privacy entitlement (Apple TN3179); TestFlight |

---

## Known blockers

| ID | Blocker | Impact | Status |
| --- | --- | --- | --- |
| B1 | No macOS/Xcode host | iOS binding path (plan §4.5) and iOS/macOS builds unverified | Open: deferred |
| B2 | No Android device attached yet | Phase 0 exit gate and all real-device work | Open |
| B3 | Second device for Android ↔ Android and Linux ↔ Linux not confirmed | Phase 2 item 4.5a, Phase 7 matrix | Open |
