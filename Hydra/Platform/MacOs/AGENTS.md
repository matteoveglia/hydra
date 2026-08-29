# macOS platform rules

This file applies to `Hydra/Platform/MacOs/` and supplements the repository root guide.

## Current fork behavior

- `MacInputHandler` captures and resolves local macOS input on the master.
- `MacOutputHandler` injects ordinary input and dispatches special actions on the slave.
- Option+Space must remain a physical Space key plus Option modifier so application shortcuts such as 1Password Quick Access work.
- System volume/mute uses CoreAudio, with the existing bounded AppleScript fallback for output devices that have not initialized virtual main volume.
- Play/pause/next/previous dynamically loads the private MediaRemote framework and retains the legacy synthetic-event fallback.
- Brightness routes through `MacBrightnessController` in this order:
  1. BetterDisplay's documented distributed-notification integration when BetterDisplay is running.
  2. private DisplayServices for displays macOS can control.
  3. raw DDC/CI for a generic external monitor.
  4. Hydra's legacy synthetic media-key path when no controller is available.

The raw DDC fallback currently uses the first compatible `DCPAVServiceProxy`; do not present it as robust multi-external-display routing until service-to-display matching is implemented and tested.

User-visible changes to these fork-specific paths must be reflected in the README or configuration reference where applicable. Keep claims scoped to the exact route tested; do not imply that a private API fallback or one monitor/output-device result proves general macOS support.

## Native interop rules

- Treat DisplayServices, MediaRemote, IOAVService, IOKit HID injection, and OSD APIs as private or unsupported unless Apple documents the exact symbol.
- Resolve private symbols dynamically and fail closed to the existing fallback. A missing framework/symbol must not terminate Hydra.
- Put stable P/Invoke declarations and native structs in `NativeMethods.cs`; keep feature policy in focused controller classes.
- Match native widths, signedness, calling conventions, struct layout, and ownership exactly. Retain managed delegates used by native callbacks for the callback lifetime.
- Balance CoreFoundation/IOKit ownership (`CFRelease`, `IOObjectRelease`) on every success and failure path.
- Do not add arbitrary sleeps to input handling. DDC timing delays must be bounded, documented, and serialized.
- Synthetic `NX_SYSDEFINED`, `CGEventPost`, and `IOHIDPostEvent` are not equivalent to genuine HID hardware on current macOS. Prove the actual receiving API before changing routes.

## Validation

At minimum for keyboard/media/brightness changes:

```bash
dotnet test Tests/Tests.csproj --configuration Release --no-restore --filter 'FullyQualifiedName~Mac'
dotnet build Hydra.sln --configuration Release --no-restore
dotnet publish Hydra --configuration Release --runtime osx-arm64 --self-contained --no-restore
```

Add deterministic unit tests for packet construction, parsing, key mapping, modifier handling, and fallback selection. Native side effects still require explicit live validation on the relevant hardware; report monitor, macOS version, output device, and which path executed.

Do not deploy or restart the installed LaunchAgent merely because publish succeeded. If deployment is requested, preserve the current binary, maintain its code-signing identity/requirements, restart through the existing LaunchAgent, and verify relay reconnection from both sides.
