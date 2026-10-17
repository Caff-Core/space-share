# Local Network File Sharing: AI Coding Agent Plan

## 1. Objective and fixed technology choices

Build an installed desktop and mobile application that transfers arbitrary user-selected files directly between devices on the same local network. Support individual files and batches without requiring an account, cloud service, or internet connection at runtime.

- Desktop: Tauri 2, React, TypeScript and Vite. Use Rust for privileged desktop operations and the desktop transfer engine. Do not use Electron or require a Node.js runtime in the shipped application.
- Mobile: React Native, Expo and TypeScript. Use Expo development builds for native networking integrations; do not constrain implementation to Expo Go.
- Repository: pnpm workspaces with shared TypeScript contracts, validation, state logic and design tokens.
- Rust and, where required, Kotlin/Swift are implementation dependencies for native capabilities. Do not assume every part of the application can be TypeScript.

This plan authorizes implementation, local builds and validation. Publishing, app-store submission and signing with production credentials are separate work.

## 2. Product scope

### Minimum viable product (MVP)

- Give each installation an editable display name and a stable local identity.
- Connect through a Quick Response (QR) code or manual endpoint plus verified identity fingerprint.
- Select one or multiple files; desktop also supports drag and drop.
- Send a batch to one recipient, sequentially, in either direction.
- Display sender identity, file names, individual sizes and total size before acceptance.
- Accept or reject the entire batch. No bytes are transferred before acceptance.
- Show per-file and batch progress, speed, cancellation and explicit results.
- Retry failed files without retransmitting completed files; retries restart the failed file from zero.
- Save successfully verified files and offer an explicit open/export action.
- Work with mobile apps open in the foreground.
- Keep a small local history of metadata and results; provide a clear-history action.

“Any file” means any regular file the operating system permits the user to select. Do not restrict by extension, transform contents or promise access to protected system files. Resource limits and available storage still apply.

### Subsequent releases

Automatic nearby-device discovery; bounded parallel file transfers; interrupted-transfer resume; folders; share-sheet integration; trusted-device management; background transfers where supported; multiple recipients.

### Excluded from the MVP

Internet relay, accounts, cloud storage, synchronization, Bluetooth transport, remote access, automatic file execution and automatic acceptance.

## 3. Assumptions and delivery order

Target Windows, macOS and Linux desktops plus Android and iOS mobile devices. These are intended targets, not a claim of immediate platform parity.

Default validation order: Windows + Android first, then the remaining platforms. If the available host cannot build those targets, implement against this architecture, validate what the host supports and report missing physical-device/build coverage explicitly. Do not replace Tauri or Expo to work around the host.

All device pairs are eventual requirements: desktop–desktop, desktop–mobile and mobile–mobile. The architecture must permit mobile devices to receive directly without a desktop acting as a relay. Both peers must be reachable; the same Wi-Fi name alone does not guarantee connectivity.

## 4. Mandatory feasibility gate

Before polishing screens, prove a minimal transfer between a Tauri desktop and a physical Expo development-build phone.

1. Confirm file selection, readable handles, streaming to disk and export/save behavior.
2. Prove a native foreground listener on mobile and an outbound client on both platforms. Test transfers with each device acting as receiver.
3. Prove authenticated encrypted connections using certificate/public-key pinning on desktop and mobile. Validate the actual transport library rather than assuming JavaScript fetch supports the required trust controls.
4. Transfer a file larger than normal memory buffers, observe memory use, and cancel midway.
5. Verify mobile–mobile feasibility with the chosen native transport, including the iOS binding path before calling the platform design complete.
6. Record library versions, maintenance/license checks, supported operating systems, permissions, limitations and a runnable proof in an architecture decision record.

Prefer established native networking and cryptography libraries. If no suitable Expo-compatible bridge exists, implement a small local Expo module. Do not change the product into a desktop-dependent mobile client without user agreement.

Exit gate: both transfer directions work on the initial desktop/phone pair with binary streaming, authenticated peer identity and integrity verification. Missing physical-device coverage remains an explicit blocker for a release claim.

## 5. Repository layout and ownership

```text
apps/
  desktop/
    src/                    React interface and desktop adapter
    src-tauri/              Rust commands, transport, storage and identity
  mobile/
    src/                    React Native interface and mobile adapter
    modules/                Local Expo modules if needed
packages/
  protocol/                 Runtime schemas, derived types, version and fixtures
  transfer-core/            Pure state machine, queue and progress calculations
  design-tokens/            Colours, spacing and typography
  test-fixtures/            Shared interoperability and malformed-input cases
docs/
  architecture.md
  protocol-v1.md
  decisions/
  platform-support.md
```

- Keep shared TypeScript packages independent of React, native file objects and platform imports.
- Share logic and tokens; allow desktop and mobile screens to use platform-specific components.
- Use runtime validation, such as Zod, at every untrusted message boundary. Derive TypeScript types from schemas.
- Rust and mobile native types must match the same versioned wire specification. Use generated bindings where practical, plus shared fixtures to detect drift. TypeScript types alone cannot enforce native interoperability.
- Keep file bytes inside native streaming paths. Do not pass whole files or base64 payloads through Tauri commands, JavaScript state or the React Native bridge.
- Native adapters expose start/stop listener, connect, offer, accept, reject, cancel and retry commands plus metadata/progress events. Throttle progress events to avoid rendering pressure.
- The shared state machine manages presentation and intent; the native engine owns authoritative byte counts, filesystem writes and transport lifecycle. Reconcile snapshots after reconnecting the interface to avoid duplicate execution.

## 6. Proposed local transfer protocol

Use Hypertext Transfer Protocol Secure (HTTPS) on an application-owned local listener. The receiver listens; the sender connects. Use persistent streaming requests for file bodies and bounded polling for offer status/progress initially. Add a WebSocket control channel only if justified.

This is a proposed architecture, subject to the feasibility gate. Record any protocol adjustment before building multiple implementations.

### Connection and trust

- Generate an installation key pair and keep private material in platform-protected storage.
- A receive session advertises its address, port, protocol version, public-key/certificate fingerprint and an expiring single-use pairing secret in its QR payload.
- Pin the peer identity before transmitting the secret or file metadata. Never globally disable certificate validation.
- Manual connection requires the same fingerprint verification and pairing secret; an address or device name alone is not authentication.
- Pairing exchanges authenticated peer identities and issues scoped, expiring session credentials using an established implementation. Do not invent a cryptographic handshake.
- Invalidate pairing secrets after use and session credentials on revocation/expiry. Keep secrets out of logs and history.
- Do not expose receive sessions indefinitely by default. Show a visible receiving state and allow stopping it.

### Protocol contract

Specify exact request/response schemas, authentication, status codes and size limits in `docs/protocol-v1.md` before implementation:

| Operation | Purpose |
| --- | --- |
| Pair/session | Establish an authenticated, expiring local session |
| Create offer | Submit a versioned manifest for recipient review |
| Read offer status | Report pending, accepted, rejected, expired or canceled |
| Accept/reject | Local recipient action updates allowed transfer state |
| Upload file | Stream one accepted file by opaque offer/file identifiers |
| Commit file | Verify byte count and digest, then return stored-file acknowledgement |
| Cancel offer | Stop active streams and clean incomplete files |
| Read result | Return authoritative per-file and batch outcomes |

- A manifest includes protocol version, offer identifier, per-file identifier, display filename and byte length. Treat media type as optional descriptive data.
- Represent byte lengths as validated decimal strings on the wire to avoid JavaScript integer precision ambiguity. Define supported maximum sizes explicitly.
- Negotiate supported versions or fail clearly. Unknown incompatible versions must not be silently accepted.
- Bind every credential and file request to the authenticated peer, accepted offer and file identifier. Prevent duplicate/concurrent uploads for the same file.
- Define timeouts, expiration, idempotency, retry identifiers and reconnect behavior. Never assume a lost acknowledgement means the receiver did not store the file.
- Sequential transfer means one active file stream per batch for the MVP. Enforce configured global receive limits as well.
- Stream a Secure Hash Algorithm 256-bit (SHA-256) digest on each peer; transmit the sender digest through an authenticated completion message. Success requires matching length, digest and receiver acknowledgement.
- Digest verification detects corruption; peer authentication provides identity. Neither replaces the other.
- Receive into an app-owned temporary file. Commit to the final location only after verification, atomically where supported. Model native document-provider exports separately where atomic rename is unavailable.

## 7. State, storage and failure behavior

Per-file states: queued → transferring → verifying → completed; nonterminal states can become failed or canceled. Waiting for acceptance is a batch state. Reject invalid transitions and define terminal-state immutability.

Batch outcomes must distinguish completed, partially completed, rejected, canceled and failed. Show completed files even when later files fail. Aggregate progress is byte-weighted; a zero-byte file completes through acknowledgement without division by zero.

- Cancel closes streams and removes unfinished temporary files; completed files remain.
- Network loss or app suspension stops the active attempt. Persist safe results, reconcile receiver state on reconnect and offer a retry. Do not promise background completion or byte-offset resume in the MVP.
- Handle permission denial, unreadable/source-changed files, storage exhaustion, address changes, endpoint timeout, app exit and hash mismatch explicitly.
- Recheck source metadata or use a stable snapshot/handle so a changing file cannot silently produce a false success.
- Check estimated storage before acceptance when possible; also handle write failures during transfer.
- Avoid silent overwrite: use collision-safe names or an explicit recipient choice.
- Reject path traversal, absolute paths, separators, reserved names and abusive filename lengths. Remote filenames never determine unrestricted filesystem paths.
- Limit manifest length, file count, metadata size, request rate and unauthenticated connection lifetime. Store limits centrally and document them.
- Clear stale partial files at startup using a bounded retention policy. History contains metadata/status only, with configurable retention and no secrets or file contents.
- Never automatically open received files. Opening requires a user action.

## 8. Interface and platform integration

Provide four primary views: nearby/connect, outgoing selection, incoming approval and active/history transfers. Include clear empty, permission-denied, disconnected and partial-failure states. Use accessible labels, keyboard navigation on desktop and screen-reader-friendly progress.

Tauri: use narrow commands and capabilities, restrict filesystem access to selected sources and configured destinations, configure a restrictive Content Security Policy (CSP), and prevent untrusted remote content from invoking privileged commands. Keep network/file work off the interface thread.

Expo: configure local-network permission messages, required native build settings and file/provider access. Native networking support must work in a production-style development build, not just a simulator. Explain the foreground requirement while transferring.

For later discovery use multicast Domain Name System (mDNS)/Bonjour and publish only minimal service metadata. Discovery is not authentication. Always retain QR/manual connection. Provide actionable errors for denied permission, firewall rules and guest-network/client isolation; never modify firewall settings silently.

## 9. Implementation milestones and completion gates

- [ ] **M0: Inspect and document.** Read repository instructions, inventory existing code, choose compatible stable versions, document defaults and create the protocol outline. Preserve unrelated work.
- [ ] **M1: Feasibility spike.** Complete section 4 and record the selected native transport and security approach. Do not proceed on an untested mobile receiving assumption.
- [ ] **M2: Monorepo foundation.** Configure workspaces, strict TypeScript, formatting, linting, shared schemas, Rust checks and application shells. Fresh installs/builds are reproducible with lockfiles.
- [ ] **M3: One secure file.** Implement pairing, offer/approval, native binary streaming, verification, durable commit and acknowledgement. Demonstrate both directions on real devices.
- [ ] **M4: Batches and failures.** Add the sequential queue, byte-weighted progress, cancellation, partial outcomes, per-file retry and reconnect reconciliation. Verify duplicate retries cannot duplicate committed files accidentally.
- [ ] **M5: Usable MVP.** Add desktop drag/drop, mobile selection/export, receiving controls, accessible states and bounded local history. Resolve platform permissions and storage behavior.
- [ ] **M6: Hardening and packaging.** Run interoperability/security tests, build desktop installers and mobile test builds on supported hosts, document platform coverage and complete an offline end-to-end demonstration.
- [ ] **M7: Post-MVP discovery.** Add automatic discovery only after the reliable manual/QR path works. Retest permissions and fallback behavior.

At each milestone, update this checklist, record evidence and unresolved limitations, and leave the repository buildable. Do not mark a gate complete using mocks alone.

## 10. Verification requirements

Automated tests must cover consequential behavior:

- Runtime schema validation, malformed manifests, incompatible versions and shared Rust/native/TypeScript protocol fixtures.
- State transitions, partial batches, retry idempotency, zero-byte files and byte-weighted progress.
- Incorrect certificate fingerprints, expired/replayed pairing secrets, unauthorized uploads and offer/file credential isolation.
- Integrity mismatch, truncated streams, disk write failure, path traversal and filename collisions.
- Cancellation cleanup and lost acknowledgement/reconnect reconciliation.

Real-device validation must include:

- Desktop–desktop, desktop–Android, desktop–iOS and Android–iOS transfers in both directions before claiming full support.
- Binary files, Unicode/long filenames, duplicate names, empty files, a many-file batch and a file of at least 1 GiB where device storage permits.
- Memory measurements during large-file transfer: record baseline, peak and sustained use; memory must not grow with total file length.
- Wi-Fi loss, receiver exit, source deletion/change, permission denial, low storage and phone background/foreground transitions.
- Transfers with internet disconnected but local networking active; no cloud endpoint may be required.

Run TypeScript checks, linting, relevant unit/integration tests, Rust formatting/checks/Clippy/tests and supported build smoke checks. Report unavailable toolchains and untested device combinations honestly. Do not invent performance numbers or successful tests.

## 11. Instructions for the implementing agent

1. Start with repository instructions and M0, then execute milestones in order.
2. Resolve routine implementation details autonomously and document tradeoffs. Ask only when a decision changes product scope, required platforms or an explicit technology choice.
3. Verify current official documentation before selecting native integrations. Pin compatible dependency versions and avoid unsupported cross-platform claims.
4. Keep transport/native interfaces small, use structured errors and redact secrets and sensitive paths from logs.
5. Do not introduce a database server, hosted signaling service or separate user-run backend. Each installed application contains its own local transfer service.
6. Keep runtime behavior offline. Development package downloads and build services are separate concerns.
7. Maintain `docs/platform-support.md` with build status, physical-device evidence, permissions and known limitations per target.
8. Finish each work report with implemented behavior, meaningful validation, remaining blockers and the next milestone.

## 12. Official references

Consult the current versions of these documents during implementation:

- Tauri process model: https://v2.tauri.app/concept/process-model/
- Tauri frontend-to-Rust commands: https://v2.tauri.app/develop/calling-rust/
- Tauri security and capabilities: https://v2.tauri.app/security/
- Expo native code and development builds: https://docs.expo.dev/workflow/customizing/
- Expo monorepos: https://docs.expo.dev/guides/monorepos/
- Expo filesystem: https://docs.expo.dev/versions/latest/sdk/filesystem/
- Apple local network privacy: https://developer.apple.com/documentation/technotes/tn3179-understanding-local-network-privacy

Architecture choices in this plan are recommendations. Platform capabilities and dependency support must be demonstrated by the feasibility spike and release validation.
