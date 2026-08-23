# Hydra coding-agent guide

## Scope and instruction precedence

This file applies to the entire repository. More specific `AGENTS.md` files override it for their subtree.

If `AGENTS.local.md` exists at the repository root, read it before performing live development or deployment. It contains private machine-specific facts and must remain untracked; never copy its contents into commits, PRs, issues, logs, or public documentation.

Before changing code, inspect the relevant implementation, its tests, and any public documentation affected by the behavior. Do not infer cross-platform behavior from one platform implementation.

## Repository identity

- `origin` is Matteo's personal fork: `matteoveglia/hydra`.
- `upstream` is the original project: `PacAnimal/hydra`.
- Personal-fork `main` intentionally contains local macOS behavior beyond upstream.
- Never overwrite personal macOS customizations while syncing upstream. Inspect the graph and reconcile explicitly.
- Push to `origin` only unless the user explicitly asks for an upstream PR. Upstream contributions should be small, generic branches with no machine-specific integration.
- A push to `main` triggers build/release workflows. Do not push, tag, publish, or run install/uninstall commands unless explicitly requested.

## Architecture map

- `Hydra/`: the .NET 10 KVM executable and composition root.
  - `Config/`: config loading, conditions, validation, and profile selection.
  - `Keyboard/`, `Mouse/`, `Screen/`: platform-neutral input and routing semantics.
  - `Platform/{MacOs,Windows,Linux}/`: native input/output, clipboard, screen, service, and OS integration.
  - `Relay/`: Hydra's encrypted peer protocol and Styx connection behavior.
  - `FileTransfer/`, `Update/`: file transport and self-update behavior.
- `Common/`: DTOs and interfaces shared by Hydra and Styx.
- `Styx/`: ASP.NET Core relay server. Treat wire/protocol compatibility as public behavior; consult `Styx.md`.
- `Tests/`: NUnit tests, arranged by production concern. `Tests/Setup/` contains integration fixtures and fakes.
- `HydraWebConfig/`: independent React 19/TypeScript/Vite config editor. See its scoped `AGENTS.md`.
- `docs/CONFIGURATION.md`: canonical user-facing configuration and source-build reference.
- `run-tests.sh`: native macOS, containerized Linux/X11, and optional real-Windows test lanes.

`Hydra/Program.cs` is the composition root. Platform services are selected there by OS and master/slave mode. Preserve registration order where startup services depend on screen or network state.

## Toolchain and style

- .NET SDK 10; projects target `net10.0` and C# 13.
- Nullable reference types, implicit usings, and warnings-as-errors are enabled.
- Follow `.editorconfig`. Do not perform unrelated formatting or broad mechanical rewrites.
- Prefer focused, explicit changes. Preserve existing interfaces and cross-platform fallbacks unless the task deliberately changes the contract.
- Native handles, callbacks, delegates, and buffers require explicit lifetime/ownership reasoning. Do not remove apparently redundant keep-alives or releases without proving ownership.
- Never commit `Hydra/hydra.conf`, relay passwords, network-config blobs, machine hostnames/IPs, signing credentials, logs, or generated binaries.
- Do not edit `bin/`, `obj/`, `test-output/`, generated MacShield app contents, or package lockfiles unless the task requires regenerating them.

## Validation

Restore once when dependencies changed or build artifacts are absent:

```bash
dotnet restore Hydra.sln
```

For ordinary .NET changes, run a focused test first, then the solution build:

```bash
dotnet test Tests/Tests.csproj --configuration Release --no-restore --filter 'FullyQualifiedName~RelevantTestClass'
dotnet build Hydra.sln --configuration Release --no-restore
```

Choose broader validation in proportion to the change:

- Platform-neutral config, relay, routing, or shared DTO changes: `./run-tests.sh mac` at minimum.
- Linux/X11 behavior: `./run-tests.sh linux` (requires Docker and runs Xvfb).
- Windows behavior: `./run-tests.sh windows` only when the configured `windows` SSH host is available.
- Cross-platform/high-risk changes: `./run-tests.sh all` when all environments are available; otherwise report unrun lanes explicitly.
- macOS packaging or native changes: additionally publish with `dotnet publish Hydra --configuration Release --runtime osx-arm64 --self-contained --no-restore`.
- Styx container changes: build `Styx/Dockerfile` locally when Docker is available.
- HydraWebConfig changes: follow `HydraWebConfig/AGENTS.md`.

Platform-gated tests can skip on the wrong OS. A green macOS run does not prove Windows or Linux native behavior. Do not claim validation for a lane that did not execute.

Before committing or handing off, run `git diff --check`, inspect `git status --short`, and summarize both validation performed and validation not performed.

## Behavior and compatibility rules

- Input changes have two halves: capture/resolve on the master and inject/execute on the slave. Trace both before patching.
- Preserve key-down/key-up pairing, modifier state, repeat semantics, and disconnect cleanup.
- Maintain Unicode/layout behavior; do not replace resolved-character routing with physical-key assumptions globally.
- Keep platform fallbacks when a native/private facility is unavailable.
- Changes to config fields require parsing/validation tests and corresponding updates to `docs/CONFIGURATION.md` and the web editor when applicable.
- Changes to shared DTOs, SignalR methods, encryption framing, or MessagePack behavior require compatibility tests and an update to `Styx.md` when the documented protocol changes.
- Avoid blocking the input/network hot path. If native work can be slow, bounded retries and explicit fallbacks are required.

## Runtime and deployment safety

- Build output is not the installed Hydra binary.
- `--install` and `--uninstall` mutate LaunchAgents/services; do not run them as routine validation.
- Do not replace a running Hydra binary, restart Hydra on another machine, alter Accessibility/TCC permissions, or change `hydra.conf` unless the user explicitly asks for deployment or live testing.
- For live KVM tests, preserve a recovery path (local input, Screen Sharing, or SSH) before restarting either endpoint.

## Definition of done

- The change is narrowly scoped and explained by the diff.
- Relevant regression coverage exists where practical.
- Required docs/config editor changes are included.
- Focused tests and the appropriate build/publish lane pass.
- Any platform or live validation gaps are stated plainly.
- The working tree contains no unintended generated files or unrelated edits.
