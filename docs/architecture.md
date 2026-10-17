# Architecture

Concrete design for the Linux + Android track. Rationale and requirements live in [`FILE_SHARING_AGENT_PLAN.md`](./FILE_SHARING_AGENT_PLAN.md) (plan §5–§8). Versions: [ADR 0001](./decisions/0001-stack-versions.md). Wire contract: [`protocol-v1.md`](./protocol-v1.md).

Status: **proposed**. Items marked _(spike)_ are confirmed or replaced by the Phase 2 feasibility spike and recorded in ADR 0002.

## 1. System overview

Every installation is a peer. It can **receive** by running a local HTTPS listener, and **send** by connecting to another peer's listener. No relay, server or cloud is involved; desktop is never required for mobile ↔ mobile.

```text
 Sender device                                   Receiver device
┌──────────────────────────┐                 ┌──────────────────────────┐
│ UI (React / React Native)│                 │ UI (React / React Native)│
│  transfer-core state     │                 │  transfer-core state     │
├────────── adapter ───────┤                 ├────────── adapter ───────┤
│ Native transfer engine   │  HTTPS (TLS 1.3,│ Native transfer engine   │
│  - outbound client       │  pinned certs,  │  - HTTPS listener        │
│  - file reader + SHA-256 │ ──mutual auth──▶│  - temp file + SHA-256   │
│  - identity / keystore   │                 │  - commit + history      │
└──────────────────────────┘                 └──────────────────────────┘
```

## 2. Layers and ownership

| Layer | Desktop | Android | Owns |
| --- | --- | --- | --- |
| UI | React + Vite in Tauri webview (`apps/desktop/src`) | React Native (`apps/mobile/src`) | Rendering, user intent |
| Shared logic | `packages/transfer-core`, `packages/protocol` | same | Presentation state machine, schemas, progress maths |
| Adapter | Tauri commands + events | Expo module (JSI/bridge) | Narrow command/event surface, nothing else |
| Native engine | Rust in `src-tauri` | Native module _(spike: shared Rust via uniffi, or Kotlin)_ | Sockets, TLS, file bytes, hashing, filesystem writes, authoritative byte counts |
| Storage | App data dir + Secret Service | App-private storage + Android Keystore | Keys, temp files, history |

Rules (plan §5):

- **File bytes never cross the adapter.** No whole files or base64 through Tauri IPC, JS state or the React Native bridge. The UI passes opaque handles (desktop paths chosen by dialog; Android content URIs) and receives metadata/progress events.
- **Native engine is authoritative** for byte counts, file state and transport lifecycle. `transfer-core` mirrors it for presentation and reconciles from engine snapshots after UI reload.
- Shared TS packages import nothing from React, Tauri, React Native or Node built-ins.
- Every untrusted boundary (network input, and adapter events entering JS) is validated with Zod schemas from `packages/protocol`.

## 3. Native engine options _(spike)_

| Option | Description | Pros | Cons |
| --- | --- | --- | --- |
| **A. Shared Rust engine (preferred)** | One `crates/transfer-engine` crate used by Tauri directly and by Android via uniffi-generated Kotlin bindings in a local Expo module | One protocol/TLS/hash implementation; no wire drift between desktop and mobile; same path serves iOS later (uniffi Swift) | Android Rust cross-compilation (NDK 27, `cargo-ndk`), larger APK, JNI/uniffi learning curve |
| B. Kotlin engine | Kotlin listener (e.g. Ktor/Netty or NanoHTTPD) + OkHttp client with a pinning `X509TrustManager` | Native Android tooling, smaller binary | Two protocol implementations to keep in sync; iOS would need a third |

Option A changes the repo layout by adding `crates/` (below). Final choice goes in ADR 0002 after both directions run on device.

## 4. Repository layout

Plan §5 layout plus a Rust workspace:

```text
apps/
  desktop/
    src/                 React UI + Tauri adapter
    src-tauri/           Tauri shell: commands, capabilities, wiring to engine
  mobile/
    src/                 React Native UI + mobile adapter
    modules/transfer/    Local Expo module wrapping the engine (Kotlin, uniffi bindings)
crates/                  (Option A) Rust workspace
  transfer-engine/       Listener, client, TLS/pinning, streaming, hashing, commit, limits
  protocol-types/        serde types matching protocol-v1, checked against shared fixtures
packages/
  protocol/              Zod schemas, derived TS types, PROTOCOL_VERSION, limits
  transfer-core/         Pure state machine, queue, byte-weighted progress
  design-tokens/
  test-fixtures/         JSON wire fixtures (valid + malformed), used by TS, Rust and Kotlin tests
docs/
```

## 5. Identity and trust

- **Installation identity:** on first run generate an ECDSA P-256 (or Ed25519, if supported by both TLS stacks; decided in the spike) key pair and a self-signed X.509 certificate (`rcgen`). Private key is stored in the Linux Secret Service (`keyring`) and the Android Keystore (or encrypted with a Keystore-held key if the TLS stack requires the raw key).
- **Fingerprint:** SHA-256 of the certificate's SubjectPublicKeyInfo (SPKI), shown as grouped hex and embedded in QR codes.
- **Transport:** TLS 1.3 only, mutual authentication. Each side presents its installation certificate; each side verifies the peer by **exact SPKI pin**, never by CA chain or hostname. Certificate validation is never globally disabled. A custom verifier accepts only the pinned key.
- **Pairing:** receiver's QR (or manual entry) carries address, port, protocol version, receiver fingerprint and an expiring single-use pairing secret. The sender pins the receiver, then presents the secret inside the TLS channel. The receiver records the sender's SPKI fingerprint and issues a scoped, expiring session token bound to that fingerprint. Only established primitives are used (TLS 1.3 + random bearer secret + random session token); no custom handshake.
- **Discovery (later, Phase 9)** only supplies candidate addresses; it never establishes trust.

## 6. Transfer flow (one batch)

1. Receiver starts a receive session → listener bound, pairing secret minted, QR shown, "Receiving" state visible.
2. Sender scans QR → TLS connect with pin → `POST /v1/pair` with secret → session token.
3. Sender reads metadata from selected handles → `POST /v1/offers` with manifest.
4. Receiver UI shows sender identity, names, sizes, total → user accepts or rejects the whole batch. No file bytes before acceptance.
5. Sender streams files **sequentially**: `PUT /v1/offers/{offer}/files/{file}` with body streamed from the source handle; both sides compute SHA-256 while streaming. Receiver writes to an app-owned temp file.
6. Sender sends `POST .../commit` with byte length + digest → receiver verifies, atomically moves to the final collision-safe name, returns acknowledgement.
7. Repeat per file; then `GET /v1/offers/{offer}/result` for authoritative outcomes.
8. Cancel at any point: `POST /v1/offers/{offer}/cancel` → streams closed, temp files deleted, committed files kept.

Endpoints and schemas: [`protocol-v1.md`](./protocol-v1.md).

## 7. Adapter contract

Same logical surface on both platforms (names illustrative; finalised in Phase 3):

| Commands (UI → engine) | Events (engine → UI, throttled ≤ 10 Hz per offer) |
| --- | --- |
| `getIdentity`, `setDisplayName` | `receiveSessionChanged` |
| `startReceiving`, `stopReceiving` | `incomingOffer` |
| `connect(qrPayload \| manualEndpoint)` | `offerStatusChanged` |
| `createOffer(peer, handles[])` | `fileProgress` (bytes, rate) |
| `acceptOffer`, `rejectOffer` | `fileStateChanged` |
| `cancelOffer`, `retryFile` | `offerResult` |
| `getSnapshot` (reconciliation) | `engineError` (structured, redacted) |
| `exportFile` / `openFile` (user action only) | |

Errors are structured `{ code, message, retryable }`. Paths, secrets and tokens are redacted from logs and events.

## 8. State model

Per file: `queued → transferring → verifying → completed`; any non-terminal state → `failed` | `canceled`. Terminal states are immutable. Batch: `draft → pending_acceptance → accepted → (in_progress) → completed | partially_completed | rejected | canceled | failed | expired`. Progress is byte-weighted; zero-byte files complete through acknowledgement. Implemented once in `transfer-core` (presentation) and enforced in the engine (authoritative).

## 9. Storage

| Data | Desktop (Linux) | Android |
| --- | --- | --- |
| Private key | Secret Service (`keyring`) | Android Keystore |
| Temp/partial files | `$XDG_DATA_HOME/space-share/incoming/.partial/` | App-private files dir `incoming/.partial/` |
| Received files | User-configured folder (default `~/Downloads/Space Share/`) | App-private storage, then explicit export via SAF/MediaStore |
| History | JSON file in app data dir, metadata only, bounded retention | Same |
| Settings | App config dir | App-private storage |

Stale partials are removed at startup per retention policy. Received files are never opened automatically.

## 10. Platform integration (this track)

- **Tauri:** minimal capabilities; no remote content; restrictive CSP (`default-src 'self'`); file access limited to dialog-selected sources and the configured destination; network and file work on async tasks, never the UI thread.
- **Android:** `INTERNET`, `ACCESS_NETWORK_STATE`, `ACCESS_WIFI_STATE`; foreground service only if required for listener stability _(spike)_; no broad storage permission (use document picker + SAF); `NEARBY_WIFI_DEVICES`/multicast lock only for Phase 9 discovery. Foreground requirement explained in the UI.
- **Firewall:** never modified silently. Fedora's `FedoraWorkstation` zone allows TCP 1025–65535 by default; other distros may need a user-applied rule, documented in `platform-support.md`.
