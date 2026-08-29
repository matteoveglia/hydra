# Hydra stability and simplification programme

Status: implementation programme; web configuration surface removed  
Baseline date: 2026-08-29  
Baseline commit: `c45e6fa58ae0f2f615716e0ef1afa6e4d8ddb00e` (`main`, equal to `origin/main`)  
Comparison base: `upstream/main` at merge base `a20e98dbf3b9052ba63808d1a36568aaf35356fa`

Implementation decision (2026-08-29): `HydraWebConfig` and its dedicated CI/container surface were removed. `hydra tui` is Hydra's sole interactive configuration editor. The canonical .NET parser and validator remain the source of truth, while the TUI guided form must preserve retained raw JSON for fields it does not expose.

## Executive decision

The generic programme is directionally correct, but its assumed scale and architecture do not fit Hydra. Hydra is a compact, cross-platform, latency-sensitive system rather than a large authenticated web product. The programme should therefore optimize for preserved input state, protocol compatibility, native resource safety, recovery, and truthful platform validation—not maximum consolidation or a large permanent audit bureaucracy.

The objective is:

> Make Hydra safer and cheaper to change while preserving input capture/injection semantics, relay compatibility, recovery paths, platform fallbacks, and the personal fork's macOS behavior.

Do not begin broad refactoring until the immediate runtime failure paths and the canonical .NET/TUI configuration boundary are characterized with focused tests.

## Review of the generic programme

Retain these parts unchanged in spirit:

- behavior-preserving invariants before structural edits;
- deterministic evidence before agent judgment;
- semantic domains rather than equal-size file shards;
- small reversible changes with independent criticism;
- commit-pinned, disposable audit artifacts;
- explicit stop, rollback, leave-alone, and unrun-validation records.

Adapt these parts for Hydra:

- replace auth/database/API/product-domain workstreams with input, screen, relay, native platform, lifecycle, management/recovery, and config-contract workstreams;
- treat the personal-fork delta against upstream as the major feature delta;
- reduce the first inventory wave from 8–24 scouts to six coherent scopes;
- use input/protocol/native golden flows rather than CRUD, permission, and screenshot flows;
- make live platform evidence and installed-runtime identity separate from compilation and publishing;
- prioritize failure-path characterization ahead of broad module decomposition.

Drop these defaults unless Hydra grows substantially:

- a permanent semantic record for every file;
- a large `.refactor-audit` tree before a candidate needs it;
- web-specific authorization, migration, and visual-regression workstreams that do not map to Hydra;
- file-length or deleted-line targets;
- automatic cleanup based on static zero-reference or advisory results.

## Current baseline

### Repository composition

- 275 tracked files remain after the web-editor removal. Of those, 243 are tracked C# files containing approximately 38,100 lines.
- Principal areas are `Hydra` (runtime and composition), `Common` (Styx DTOs/constants), `Styx` (relay server), and `Tests`.
- The largest production files are `Hydra/Tui/HydraTui.cs` (1,471 lines), `Hydra/Screen/InputRouter.cs` (1,415), `Hydra/Platform/Linux/XorgKeyResolver.cs` (1,305), `Hydra/FileTransfer/FileTransferService.cs` (815), and several platform-native interop/handler files.
- File length is only a routing signal. Native declarations, key maps, and cohesive protocol tables must not be split merely to reduce line counts.

### Personal-fork delta

The meaningful feature delta is the personal fork's current `main` against `upstream/main`, not an unmerged feature branch. It is 19 commits ahead and changes 84 files with approximately 7,786 additions and 272 deletions. The dominant additions are:

- local and paired remote TUI management;
- transactional config apply/rollback and lifecycle control;
- relay route selection and diagnostics;
- macOS media, audio, and brightness behavior;
- input/file-transfer latency changes;
- their associated tests and documentation.

This delta should receive a dedicated audit because it is both recent and intentionally fork-specific. Future upstream synchronization must preserve the fork's macOS behavior and reconcile the graph explicitly.

### Verification snapshot

- `dotnet test Tests/Tests.csproj --configuration Release --no-restore`: 773 passed, 8 skipped, 0 failed (781 total). The skips include platform-dependent clipboard, key-resolution, and path-normalization cases.
- `dotnet build Hydra.sln --configuration Release --no-restore`: passed with 0 warnings and 0 errors.
- Debug solution build and `dotnet format Hydra.sln --no-restore --verify-no-changes` also passed during the audit.
- NuGet's current vulnerability check reported no known vulnerable packages.
- The working tree was clean before this document was added. No runtime, installed binary, private config, service, or remote machine was inspected or changed.

The retired web editor's test, lint, coverage, dependency-advisory, and dead-code findings are retained only in the audit record that motivated its removal; they are no longer active repository gates or backlog items.

### Validation gaps

- No Linux/X11 container lane, real Windows lane, macOS self-contained publish, Styx container build, installed-runtime check, live two-machine KVM flow, or hardware-specific media/brightness path was executed for this audit.
- Current CI tests .NET on Linux/X11 and Windows, but has no macOS test job. Its macOS job publishes artifacts without executing the Mac-native suite.
- TUI behavior is protected mainly through model/helper and management-service tests; there is no broad deterministic Terminal.Gui interaction harness.

## Hydra-specific behavioral invariants

Every candidate must state which of these invariants it preserves:

1. Key-down/key-up, modifier, button-down/button-up, repeat, and disconnect cleanup remain paired.
2. Capture/resolve on the master and inject/execute on the slave remain compatible for every supported platform pair.
3. Unicode/layout-aware routing remains the default; physical-key routing is retained only for deliberate cases such as Option+Space.
4. Input/control traffic remains non-blocking and prioritized over bulk work; reliable release events are never dropped.
5. Styx and Hydra message framing, casing, ordering, size limits, authentication, and backward-compatible defaults remain stable unless explicitly versioned.
6. Local management IPC remains local-only. Paired remote management keeps separate credentials, freshness/replay checks, redaction, revision checks, connectivity guards, and automatic rollback.
7. Missing private/native facilities fail into existing bounded fallbacks rather than terminating Hydra.
8. Native delegates, handles, buffers, CoreFoundation/IOKit objects, COM objects, and callbacks retain explicit lifetime and release ownership.
9. Configuration written by the TUI is accepted by Hydra's canonical parser and validator and does not silently lose supported, unknown, or advanced fields retained in the source JSON.
10. A build or publish artifact is not treated as the installed runtime; deployments preserve code-signing identity, config, service ownership, and a recovery path.

## Audit operating model

The generic recommendation of 8–24 initial scouts is excessive for this repository. Use five semantic scopes, with at most three workers active beside the orchestrator:

1. Composition, config resolution, lifecycle, local/remote management, and TUI.
2. Input routing, screen transitions, keyboard/mouse state, activity, and dormancy.
3. Relay client, Hydra message protocol, Styx server, encryption, latency, and file-transfer interaction.
4. macOS, Windows, and Linux native capture/injection, clipboard, screen, service, and fallback behavior.
5. Tests, fixtures, CI, release/publish, documentation, and fork/upstream delta.

After that inventory, use focused cross-cutting reviews for configuration contracts, protocol compatibility, input-state ownership, native lifetime, duplicate geometry/rules, and test/CI gaps. An independent critic should review every candidate rated high risk.

For Hydra's current size, a concise domain inventory is preferable to a permanent record for every file. Generate file-level JSONL only for a high-risk domain or a future substantially larger delta. Pin every audit artifact to a commit and regenerate rather than manually maintaining it.

## Adapted phases

### Phase 0 — Restore and record a green baseline

1. Record exact local and CI commands, platform skips, known warnings, and current dependency advisories.
2. Add golden-flow definitions and recovery requirements before structural edits.
3. Record both the current commit and the `upstream/main...main` fork delta.

Golden flows must include config load/profile selection; master/slave connect and reconnect; screen crossing and held-input cleanup; ordinary, modified, repeated, and Unicode typing; clipboard round-trip; file transfer and cancellation; local TUI status/config/control; paired remote fetch/apply/confirm/rollback; screensaver/lock behavior; TUI guided edits accepted by the .NET parser without losing retained fields; and platform-specific media/brightness fallbacks where applicable.

### Phase 1 — Deterministic composition and delta map

Produce a compact external audit artifact containing:

- tracked source/test/doc/tooling composition and exclusions;
- largest and highest-churn files;
- project and namespace dependency map;
- explicit DI/lazy dependency cycles;
- the personal-fork delta against upstream;
- configuration-contract field/rule matrices across .NET, the TUI, docs, and examples;
- protocol/message compatibility matrix;
- current platform/CI validation matrix.

Do not inventory `bin`, `obj`, `dist`, `node_modules`, `test-output`, generated MacShield app contents, lockfiles as semantic source, or opaque assets.

### Phase 2 — Domain and cross-cutting audit

For each of the five domains, record entry points, owners, side effects, public contracts, paired tests, unexpected dependencies, high-churn risks, and evidence-backed candidates. Run these cross-cutting checks:

- config model/validation/round-trip consistency;
- input and screen state ownership;
- duplicated geometry, routing, and transformation rules;
- relay and management protocol compatibility;
- native handle/callback ownership and fallback selection;
- dead/reachable code with dynamic/reflection/config consumers considered;
- error, retry, cancellation, and shutdown behavior;
- CI coverage versus claimed platform support.

### Phase 3 — Candidate registry and pilot

Rank by defect evidence, change frequency, blast radius, test protection, confidence, and migration risk. The first implementation pilot should be the bounded Linux Xlib initialization repair, followed by the remaining readiness and drain failure paths as separately reviewed changes. Configuration hardening remains an early source-of-truth task, now confined to .NET, TUI preservation, examples, documentation, and tests.

The pilot sequence is:

1. Add fixtures that encode supported root/profile fields, defaults, aliases, invalid cases, topology semantics, and retained unknown fields.
2. Prove those fixtures against the canonical .NET parser/validator and the TUI's patch-in-place guided edit path.
3. Correct the smallest confirmed canonical validation gaps without redesigning the config system.
4. Add the focused contract tests to the existing .NET CI gate.
5. Independently review lost fields, changed defaults, secret handling, mirror expansion, and generated JSON determinism.

### Phase 4 — Bounded implementation waves

- Wave A: remove the retired web surface, retain a green baseline, and characterize and repair confirmed runtime hazards.
- Wave B: local simplifications behind stable interfaces, beginning with a deterministic TUI interaction seam.
- Wave C: source-of-truth protection through config contract tests and explicit state owners. Keep canonical parsing and validation in .NET.
- Wave D: boundary repair such as composition-root registration modules or an explicit replacement for the lazy ActivityTracker/relay dependency, only after characterization tests.
- Wave E: optional deep changes only when recurring defects or measured latency/reliability evidence proves local corrections insufficient.

## Preliminary candidate backlog

### HSS-000A — Initialize Xlib before every Linux X11 call

Status: implemented locally 2026-08-29; Linux/Xvfb and real-Xorg runtime validation remain open  
Priority: immediate runtime fix  
Confidence: high from source; requires Linux runtime validation  
Risk: moderate

When profile selection uses a `screenCount` condition, `Program.GetScreenCount()` could call `XOpenDisplay` before any platform service called `XInitThreads`; `NetworkWatcher` can later reuse the same provider. `XlibRuntime` now owns one lazy, process-wide initialization gate and every direct display open flows through it. Profile probing retains its one-screen fallback when initialization or display opening is unavailable; Xorg services fail explicitly when thread support cannot be initialized. The focused gate tests pass, but Docker was unavailable for the Linux/Xvfb lane and a real Xorg session remains unrun.

Invariant: every Xlib call occurs after successful process-wide thread initialization, without changing headless/evdev selection.

### HSS-000B1 — Make event-tap startup success explicit

Priority: immediate characterization and repair  
Confidence: high  
Risk: moderate-high

`ILocalEventTap.StartEventTap()` cannot report a typed success/failure even though macOS and Windows handlers internally distinguish failed setup, callers discard that status, and Xorg ignores some registration return codes. Add failed-start fakes for master and slave callers, platform seams for native creation/registration failure, a typed result, and truthful management health. Keep callback non-blocking and platform restart/fallback behavior explicit.

Invariant: Hydra must not appear healthy while input capture is absent.

### HSS-000B2 — Give screen-detector readiness a terminal policy

Priority: immediate characterization and repair  
Confidence: high in the wait path; exact host behavior requires verification  
Risk: moderate-high

`ScreenDetector.Get()` waits for readiness that is completed only after a successful `Detect`; a native exception before first success has no local terminal result. Add tests for first-detection failure, caller cancellation/deadline, successful retry, and change notification. Choose an explicit retry policy or fault/cancel result after verifying the hosting base class rather than hiding the decision in each platform adapter.

Invariant: native detector failure must not leave startup waiters blocked forever or fabricate a valid screen snapshot.

### HSS-000C — Keep the output drain alive after platform-action failures

Priority: immediate characterization and repair  
Confidence: high  
Risk: high because this is the injection path

`CoalescingOutputWrapper.Drain` catches `InvalidOperationException` around both collection enumeration and platform action invocation. A platform action throwing that exception can silently terminate the drain and drop subsequent key/button/mouse events; disposal also joins the worker without a timeout. Separate queue-completion handling from action failure, define fatal-versus-continue behavior, add failure/block injection tests, and expose the fault. A bounded shutdown needs cooperative ownership; do not merely time out and dispose the inner output while its worker may still be using it.

Invariant: control events remain ordered ahead of coalesced movement, release events are not silently abandoned, and shutdown is bounded even when a native action blocks.

### HSS-000D — Guarantee Windows mouse-setting recovery

Priority: immediate design/test seam; implementation requires real Windows evidence  
Confidence: high from source  
Risk: high

Relative mouse injection temporarily changes global Windows acceleration/speed, ignores restore return values, and can be interrupted by a crash or forced termination. Add a recoverable settings owner, checked restore results and telemetry, failure-injection coverage, and a real-Windows recovery test before changing the implementation. Do not treat cursor restoration as equivalent to mouse-setting restoration.

Invariant: Hydra never leaves user-global mouse settings flattened after normal stop, initialization failure, restart, or any recoverable fault path.

### HSS-001 — Protect the canonical configuration contract

Priority: immediate  
Confidence: high  
Risk: moderate

The canonical .NET validation still needs explicit decisions and regression tests for empty profile lists, multiple relay fields, empty SSIDs, missing profile names, and mode-mismatched fields. `Program` must not index the first profile after validation unless validation proves one exists.

The TUI guided editor already patches a retained JSON document rather than serializing an expanded runtime graph. Add fixtures for no-op and targeted edits containing advanced and unknown fields, mirrors, ranges, secrets, root diagnostics, and profile overrides. Prove that the guided path changes only the requested field and that the result is accepted by `HydraConfigFile.Parse` and `HydraConfig.Validate`.

Invariant: the TUI must preserve canonical semantics and retained source fields without serializing mirror-expanded runtime state or exposing secrets.

### HSS-002 — Remove the retired web configuration surface

Status: completed 2026-08-29  
Risk: low repository risk; deliberate product-scope change

The independent React/Vite editor, its package and container files, its scoped agent guidance, and its dedicated GitHub Actions workflow were removed. Repository guidance and the TUI architecture record now identify `hydra tui` as the sole interactive editor. This also closes the old web lint, unreachable-component, duplicated-geometry, and npm-advisory candidates by removing their owning product surface rather than repairing code that is no longer needed.

Invariant: removing the web surface does not remove or weaken `HydraConfigFile.Parse`, `HydraConfig.Validate`, source JSON retention, configuration documentation, or offline TUI editing.

### HSS-005 — Add a deterministic TUI interaction seam, then split the controller

Priority: early-to-middle  
Confidence: high that responsibilities are concentrated  
Risk: moderate

`HydraTui.cs` combines seven-tab construction, local/remote operations, config editing, refresh/render loops, lifecycle command polling, formatting, and Terminal.Gui widget state. Add characterization around command state transitions and a supported fake/test application driver first. Then extract cohesive internal units such as remote-config workflow, config editor binding, status formatting, and lifecycle command coordination while keeping the management protocol unchanged.

### HSS-007 — Characterize InputRouter before any structural change

Priority: investigation first  
Confidence: high that it is a hotspot; low that splitting alone helps  
Risk: high

`InputRouter.cs` is both the largest core runtime file and the highest-churn production file. Map its serialized command queue, held-state ownership, screen transitions, mouse throttling/coalescing, file-transfer hotkeys, locks, and disconnect cleanup. Add missing sequence tests before considering extraction. Do not split the state machine across independently synchronized services.

### HSS-008 — Reduce composition-root coupling only behind registration-order tests

Priority: later  
Confidence: medium  
Risk: high

`Program.cs` is a high-churn composition root with dense OS/mode registration. Extract platform/mode registration modules only if service-graph tests can prove the selected implementations, singleton aliasing, and hosted-service order. Preserve early config recovery, screen-detector startup order, session-child behavior, and Mac shield initialization.

### HSS-009 — Revisit the explicit lazy relay/activity dependency

Priority: later  
Confidence: medium  
Risk: moderate-high

The composition root explicitly enables `Lazy<T>` resolution to break the ActivityTracker/relay dependency. Map the actual ownership cycle and callback timing. Prefer an explicit event/sender boundary only if it reduces coupling without moving input work onto a slower path.

### HSS-010 — Align CI with supported platforms and local policy

Priority: early  
Confidence: high  
Risk: low-to-moderate operational cost

Add a Mac-native test job, retain Linux/X11 and real Windows CI, run the documented Release configuration and formatting check, and define when macOS publish, Styx Docker build, and cross-target publishes are required. Hardware-only brightness/media/input behavior remains a named manual lane; compilation must never be reported as runtime validation.

### HSS-011 — Centralize atomic private-file replacement

Priority: middle  
Confidence: high  
Risk: moderate

`TransactionalConfigStore`, `RemoteApplyStore`, and `RemoteManagementStore` each implement related temporary-write, flush, permission, replace/move, and cleanup sequences. Extract a low-level primitive only after symlink, permission, concurrent-writer, crash, and replacement-policy tests establish where the semantics intentionally differ.

Invariant: config revisions, private permissions, last-known-good recovery, and secret sidecars retain their current atomicity and recovery behavior.

### HSS-012 — Make embedded Styx readiness an explicit composition contract

Priority: early characterization  
Confidence: high that the race is structurally possible; runtime impact requires evidence  
Risk: moderate

The embedded server exposes `WaitForReady`, but the composition root relies on hosted-service registration order and does not await listener readiness before relay startup. Add a contract test for initial embedded connection and decide whether startup should await readiness or whether reconnect is the intentional mechanism. Preserve failure recovery and avoid blocking the input hot path.

### HSS-013 — Repair wire/domain boundary leakage only after protocol fixtures

Priority: later  
Confidence: high that layering is mixed; benefit requires a bounded pilot  
Risk: high

Platform output and screen abstractions consume Relay message records directly, and the Relay message module spans input, screen, clipboard, file transfer, diagnostics, and management. Introduce neutral commands/snapshots only through compatibility adapters after byte-value, JSON/MessagePack, ordering, and older-payload fixtures exist. This is boundary debt, not proof that the current wire contract should be redesigned.

### HSS-014 — Clear or mark stale relay latency state

Priority: early local fix  
Confidence: high  
Risk: low

Disconnect clears peers and pending probes but retains historical latency states returned by management snapshots. Add disconnect/reconnect tests and choose an explicit clear or stale-marker policy so the TUI does not present old metrics as current.

### HSS-015 — Separate WorldState responsibilities behind immutable snapshots

Priority: later  
Confidence: high that responsibilities are mixed  
Risk: high

`WorldState` contains separate internal master/slave state objects plus peer screens, encryption keys, and logger dictionaries, but departure pruning spans those stores outside one atomic update and some snapshot operations shallow-copy mutable screen lists. First add concurrency, departure, and snapshot-immutability tests. Consider broader owner extraction only if those tests or recurring change evidence justify it; do not change relay callbacks in the same step.

## Leave-alone list unless new evidence appears

- Platform-specific native implementations and fallback ladders that merely look duplicated.
- `NativeMethods` declaration blocks, key maps, and protocol tables when they are cohesive and ownership-correct.
- Mac media, audio, brightness, BetterDisplay, DisplayServices, DDC, and legacy fallbacks without live path-specific evidence.
- Relay ordering, coalescing, the intentionally unbounded ordered control queue, and reliable release/control delivery without measured performance evidence. Measure and stress outage growth before changing queue policy.
- Styx/Hydra message kinds, MessagePack/JSON casing, crypto framing, persisted sidecars, and remote-apply markers without a versioned migration plan.
- The local-only management transport boundary.
- Installed runtime, LaunchAgents/services, private config, signing identity, and remote-machine state during ordinary refactor validation.
- `Program.cs` registration order and the single serialized InputRouter state machine until characterization tests make extraction safe.

## Verification gates by candidate type

All changes: focused tests, Release solution build, `git diff --check`, and clean working-tree review.

- Config/TUI: canonical .NET fixture acceptance, guided-edit preservation tests, examples/docs, deterministic round-trip checks, and focused TUI interaction tests.
- Input/routing/shared DTOs: focused sequence/compatibility tests plus `./run-tests.sh mac`; Linux/X11 and Windows lanes when affected.
- macOS native/package changes: Mac-focused tests, Release build, self-contained `osx-arm64` publish, and explicit live hardware validation when behavior changes.
- Styx/protocol: wire-format and integration tests, `Styx.md` review, and local Styx Docker build.
- TUI: management/model tests, deterministic terminal-driver tests, publish, and smoke checks in representative local/SSH terminals. Never restart an installed Hydra instance as routine validation.
- Dependency maintenance: restore, tests, build, relevant container builds, and advisory recheck.

## Success measures

Use a small set of decision-supporting measures:

- zero known config-contract mismatches in the fixture matrix;
- green required lint/build/test lanes;
- number of platform lanes actually executed versus skipped;
- unresolved high-confidence candidates;
- change blast radius for representative config, input, relay, and TUI changes;
- protocol and input-state regressions;
- TUI/controller responsibilities covered by deterministic tests;
- time needed to locate the owning module for a requested change.

Do not set a line-deletion target. Do not make a permanent file-by-file commentary system. Do not let static dead-code, complexity, or vulnerability tools authorize changes on their own.

## Recommended first three changes

1. Land the Xlib initialization gate with focused source tests, the Linux/Xvfb lane, and an explicitly named real-Xorg validation gap if that environment is unavailable.
2. Land event-tap readiness, screen-detector readiness, and output-drain failure/shutdown handling as separate changes with their own characterization and platform gates. Schedule Windows global-setting recovery only with a real Windows verification path.
3. Add canonical .NET/TUI configuration fixtures, then correct confirmed validation gaps while proving that guided edits preserve unsupported, advanced, and unknown source fields.

Only after these changes should the programme pilot TUI decomposition or InputRouter/composition-root boundary work.
