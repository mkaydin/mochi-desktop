# KDE Plasma Wayland Compatibility Plan

## Status

- Target branch: `feature/kde-wayland-support`
- Plan baseline: September 2026
- Application version at analysis: `0.3.0a1`
- Primary KDE target: Plasma 6 on Wayland
- Existing desktop target: Fedora, GNOME, Wayland with XWayland compatibility

## 1. Objective

Add first-class KDE Plasma Wayland support without replacing Mochi's GTK4 application architecture or regressing GNOME support.

The KDE version must support the complete product surface where KDE exposes an appropriate public API:

- Native transparent desktop pet
- Movement and edge roaming
- Context, developer, and emote menus
- Focus sessions
- Speech bubbles, nameplates, and progress overlays
- MPRIS media and music reactions
- Battery, network, session, lock, suspend, and resume awareness
- Downloads and file-browsing activity
- Coarse active-application context
- Global shortcuts
- Reduced typing detection through supported accessibility APIs
- Safe persistence across outputs, scales, and display backends
- Clean installation, update, and removal on GNOME and KDE

The implementation must not add typed text, key values, file paths, URLs, or raw window titles to logs or D-Bus signals.

## 2. Current architecture assessment

Mochi is a Python 3.11+ GTK4 desktop application using PyGObject, Cairo, GDK gestures, GLib, and D-Bus. The sprite, animation, state, care, Focus, sound, and persistence subsystems are largely desktop-independent.

Important current modules include:

| Area | Current implementation |
|---|---|
| Application shell | `src/mochi/app.py` |
| Display backend selection | `src/mochi/main.py` |
| Window placement | `src/mochi/windowing.py` |
| X11/XWayland movement | `src/mochi/x11.py`, `src/mochi/x11_buddy.py` |
| Pet and input coordination | `src/mochi/buddy.py` |
| Shared state machine | `src/mochi/state.py`, `src/mochi/state_controller.py` |
| GNOME helper lifecycle | `src/mochi/helper_connection.py` |
| Typing activity | `src/mochi/typing_activity.py` |
| User presence | `src/mochi/presence_activity.py` |
| File activity | `src/mochi/file_activity.py` |
| MPRIS media and music | `src/mochi/media_activity.py`, `src/mochi/music_activity.py` |
| Battery, network, app category | `src/mochi/presence/signals.py` |
| Session, lock, and suspend | `src/mochi/presence/session.py` |
| Configuration | `src/mochi/config.py` |
| GNOME extension | `gnome-extension/mochi-typing@miflow13/` |
| Installation | `install.sh`, `scripts/install-typing-extension.sh` |

### GNOME-specific behavior

The current GNOME Shell extension provides:

- Anonymous typing pulses
- User idle and active transitions
- Focused file-manager state
- Focused YouTube state
- Coarse application category
- Developer-menu shortcut requests
- Emote-catalogue shortcut requests

The extension also uses Mutter-specific APIs such as:

- `global.backend`
- `global.display`
- Mutter idle monitors
- Meta window identity methods
- GNOME Shell keybinding APIs
- GNOME GSettings

These APIs are not available in KWin and must not become dependencies of the shared application core.

## 3. Existing cross-desktop support

The following functionality should remain shared instead of receiving KDE-specific rewrites:

| Capability | Current source | KDE status |
|---|---|---|
| Pet rendering | GTK4 and Cairo | Compatible |
| Native Wayland overlay | `gtk4-layer-shell` | Suitable for KWin |
| MPRIS media | `src/mochi/media_activity.py` | Cross-desktop |
| Music detection | `src/mochi/music_activity.py` | Cross-desktop |
| Suspend, lock, and session | `src/mochi/presence/session.py` | Cross-desktop through logind |
| Battery | `src/mochi/presence/signals.py` | Usually available through UPower |
| Network | `src/mochi/presence/signals.py` | Works with NetworkManager |
| Downloads activity | `src/mochi/file_activity.py` | Cross-desktop fallback |
| Typing fallback | AT-SPI in `src/mochi/typing_activity.py` | Partial KDE support |
| Sound | PipeWire, PulseAudio, and GStreamer | Compatible |
| State and animation | Shared Python modules | Keep desktop-neutral |
| Configuration | XDG JSON | Requires placement migration |

PowerDevil, KWallet, Plasma notifications, BlueZ, and StatusNotifierItem should not be added solely for KDE compatibility. Mochi currently has no feature that requires these services.

## 4. Current KDE blockers

### 4.1 Installer assumes GNOME

`install.sh` always invokes `scripts/install-typing-extension.sh`. That script requires `gnome-extensions` and can fail when Mochi is installed in KDE.

The installer must be separated into:

- Common application installation
- GNOME helper installation
- KDE and KWin helper installation
- Optional XWayland compatibility

A non-GNOME session must never invoke the GNOME extension installer.

### 4.2 Incorrect X11 fallback assumption

`src/mochi/app.py` selects `PresenceX11Buddy` whenever layer-shell is not enabled. This is invalid for native Wayland without layer-shell because the GDK display remains Wayland while the helpers in `src/mochi/x11.py` require X11.

Correct selection rules:

```text
Wayland + layer-shell              -> LayerShellWindowBackend
X11 or explicitly requested XWayland -> X11WindowBackend
Wayland without layer-shell         -> WaylandToplevelWindowBackend
```

`PresenceX11Buddy` must never be selected solely because `layer_shell_enabled` is false.

### 4.3 Incomplete layer-shell output support

The current layer-shell implementation does not explicitly select a monitor with `Gtk4LayerShell.set_monitor()`. Margins are output-local, so this can cause:

- Incorrect monitor restoration
- Cross-monitor drag failures
- Off-screen placement after monitor removal
- Problems with negative monitor origins
- Problems with mixed-DPI layouts
- Incorrect migration from X11 coordinates

### 4.4 X11-oriented menus and placement

`src/mochi/menu_window.py` uses X11 root coordinates and was designed around an XWayland buddy. Native Wayland surfaces skip that manual placement and rely on compositor placement.

The following surfaces need explicit native-Wayland behavior:

- Context menu
- Developer menu
- Emote catalogue
- Focus window
- Speech bubbles
- Nameplates
- Bond progress overlays

### 4.5 Incomplete placement persistence

`src/mochi/config.py` persists only global `x` and `y` values. It does not persist:

- Coordinate space
- Output identity
- Output fallback geometry
- Scale
- Backend type
- Rotation

X11 root coordinates cannot be reused directly as layer-shell margins.

### 4.6 Missing KDE global shortcuts

The developer-menu and emote-catalogue shortcuts are emitted only by the GNOME extension. Plasma support should use the XDG Global Shortcuts portal, which the KDE portal backend maps to KDE's global-shortcut infrastructure.

### 4.7 Missing KDE active-window provider

Application category, focused file-manager state, and focused browser-video state depend on GNOME Shell APIs. KWin exposes supported active-window data that can produce equivalent semantic signals.

## 5. Support policy

Supporting every historical KDE release with the same guarantee is not realistic because KWin, KDE Frameworks, xdg-desktop-portal, and xdg-desktop-portal-kde have changed across Plasma generations.

The project should support all versions through capability detection while assigning release guarantees according to qualification.

| Target | Proposed tier | Required validation |
|---|---|---|
| Plasma 5.27 Wayland | Best effort | Separate KF5 helper and portal fallback; no release guarantee |
| Plasma 6.0-6.3 | Tier 2 | Oldest supported Plasma 6.x and representative intermediate release |
| Plasma 6.4-6.7 | Tier 1 | Full Plasma Wayland release gate |
| Plasma 6.8 beta | Canary | Automated smoke tests only until stable qualification |
| Plasma 6.8 stable and later | Tier 1 after qualification | Normal release gate before promotion |
| Plasma X11 and XWayland | Secondary | Preserve X11 behavior and regression-test it |

As of September 2026, Plasma 6.7 is the current stable line and Plasma 6.8 is in beta.

Runtime diagnostics should record:

- Plasma version
- KWin version
- KDE Frameworks version when available
- GTK and GDK versions
- `gtk4-layer-shell` version
- xdg-desktop-portal version
- xdg-desktop-portal-kde version
- Session type
- GDK display backend
- Active provider and capability status
- Output configuration and scale

The application must use feature detection rather than branching primarily on version strings.

## 6. Target architecture

Introduce desktop-neutral capability interfaces:

```text
DesktopIntegration
├── SessionCapabilities
├── WindowBackend
├── ActivityProvider
├── TypingProvider
├── GlobalShortcutProvider
└── ProviderLifecycle
```

Planned implementations:

```text
WindowBackend
├── LayerShellWindowBackend
├── X11WindowBackend
└── WaylandToplevelWindowBackend

ActivityProvider
├── GnomeShellActivityProvider
├── KWinActivityProvider
├── LogindIdleProvider
├── DownloadsActivityProvider
└── CompositeActivityProvider

GlobalShortcutProvider
├── GnomeShortcutProvider
├── GlobalShortcutsPortalProvider
└── KWinLegacyShortcutProvider

TypingProvider
├── GnomeTypingProvider
├── AtspiTypingProvider
└── ReducedKdeTypingProvider
```

Provider selection must:

1. Detect the actual session and GDK backend.
2. Detect available protocols and services.
3. Select the highest-quality available provider.
4. Publish capabilities and degraded states.
5. Fall back without crashing or leaving stale state.
6. Tear down signals, timers, requests, and sessions idempotently.

Desktop-specific branches must not spread through `Buddy`, animation, care, Focus, or speech logic. Those subsystems should continue consuming semantic events.

## 7. Implementation phases

## Phase 0: Support contracts and baseline

### Goal

Prevent KDE support from being defined as merely opening the pet window.

### Tasks

1. Inventory every GNOME-provided capability.
2. Define a KDE provider or reduced fallback for every capability.
3. Add an integration status model:

```text
available
degraded
unavailable
unsupported-session
backend-error
```

4. Extend debug logging with:

   - Selected GDK backend
   - Layer-shell status
   - Window backend
   - Active-window provider
   - Typing provider
   - Global-shortcut provider
   - Idle provider
   - System-service availability
   - Output-placement backend

5. Capture current GNOME behavior before refactoring.
6. Add baseline tests for existing GNOME and XWayland behavior.

### Files

- `src/mochi/main.py`
- `src/mochi/app.py`
- `src/mochi/helper_connection.py`
- `src/mochi/presence/integration.py`
- `tests/test_main.py`
- `REGRESSION_WATCHLIST.md`

### Exit criteria

- Every GNOME capability has a KDE equivalent or documented reduced mode.
- Diagnostics contain no private desktop information.
- GNOME behavior has a repeatable regression baseline.

## Phase 1: Capability-based window backends

### Goal

Make native KDE Wayland a first-class path and prevent invalid X11 fallback selection.

### Window backend contract

A window backend should provide:

- Availability
- Surface type
- Keep-above behavior
- Initial placement
- Current position
- Movement
- Drag lifecycle
- Output selection
- Output-change handling
- Shutdown

### Layer-shell implementation

Retain the existing top-layer, non-exclusive, non-keyboard configuration and add:

1. `Gtk4LayerShell.set_monitor()` before the first surface map.
2. Explicit output selection based on saved placement.
3. Conversion from logical placement to output-local margins.
4. GDK monitor-added, monitor-removed, and monitor-change handling.
5. Remap and monitor-reselection after hotplug.
6. Recovery when the saved output disappears.
7. Explicit scale and coordinate-space detection.

### Dragging

Use intra-output layer-shell margin movement as the safe default.

For cross-output dragging:

1. Accumulate pointer-relative drag movement.
2. Detect an intentional output boundary crossing.
3. Remap the layer surface to the target output.
4. Recalculate output-local margins.
5. Resume or finish the gesture without a position jump.

Avoid remapping on every pointer event because that can interrupt the drag sequence.

### X11 compatibility

Retain `PresenceX11Buddy` only when:

- The actual GDK display is X11, or
- The user explicitly requests XWayland compatibility

Native Wayland code paths must not call X11 movement functions.

### Files

- `src/mochi/windowing.py`
- `src/mochi/app.py`
- `src/mochi/buddy.py`
- `src/mochi/x11_buddy.py`
- `src/mochi/edge_roam.py`
- `tests/test_windowing.py`
- `tests/test_main.py`

### Exit criteria

- Native KDE Wayland never invokes `XMoveWindow` or `XQueryPointer`.
- Mochi opens on the intended output.
- Monitor removal, reconnection, and mixed-scale layouts recover correctly.
- Existing GNOME XWayland behavior remains unchanged.

## Phase 2: Native Wayland menus and surfaces

### Goal

Make all Mochi-owned surfaces work under KWin without X11 coordinates.

### Menus

Refactor `src/mochi/menu_window.py` into:

- Native Wayland popover strategy
- X11 root-coordinate strategy

Prefer a `Gtk.Popover` for the main context menu on native Wayland. Keep transient GTK windows where a popover is unsuitable, but define an explicit placement policy.

Requirements:

- Correct positioning on every output
- Correct positioning at screen edges
- Mixed-DPI behavior
- Reliable menu closure
- No stale pointer or keyboard grab
- Drag works after menu closure
- Repeated menu opening does not accumulate resources

### Focus and auxiliary surfaces

Review and adapt:

- `src/mochi/presence/focus_session.py`
- `src/mochi/presence/emote_catalogue.py`
- `src/mochi/presence/bubble.py`
- `src/mochi/presence/nameplate.py`
- `src/mochi/presence/bond_progress_overlay.py`

For each surface, define whether it is:

- Anchored to Mochi
- Centered by the compositor
- Positioned within the current output
- Hidden when Mochi is not visible

### Test cases

- 100%, 125%, 150%, 175%, and 200% scale
- Mixed-DPI monitors
- Negative monitor origins
- Portrait and rotated outputs
- Popups at every output edge
- Monitor removal while a popup is open

### Exit criteria

- Context menu, catalogue, Focus, bubbles, nameplates, and overlays work on Plasma Wayland.
- No native Wayland surface falls back to X11 root coordinates.

## Phase 3: Placement persistence and migration

### Goal

Make saved placement portable across backends, desktop environments, monitors, and scale configurations.

### New placement schema

```json
{
  "schema_version": 2,
  "position": {
    "coordinate_space": "layer-shell",
    "x": 120,
    "y": 80,
    "output_id": "DP-1",
    "output_fallback_geometry": [0, 0, 2560, 1440],
    "fallback_monitor_index": 0
  }
}
```

### Output identity priority

1. Stable Wayland output name where available
2. GDK connector
3. Manufacturer and model with disambiguating index
4. Nearest valid output
5. Primary output

### Migration rules

- Preserve old `x` and `y` fields during migration.
- Match old coordinates to a current X11 output when possible.
- Convert matching X11 positions into output-local layer-shell margins.
- Never directly reuse X11 root coordinates as layer-shell margins.
- Select the nearest output when output identity is missing.
- Clamp placement to valid output bounds.
- Move to the nearest output when the saved output disappears.
- Preserve bond, Focus, sound, and unrelated configuration.
- Roll back safely if a migrated position cannot be restored.

### Files

- `src/mochi/config.py`
- `src/mochi/windowing.py`
- `tests/test_config.py`
- `tests/test_windowing.py`

### Exit criteria

- GNOME and KDE can share the same config.
- Switching GNOME XWayland and KDE Wayland does not lose Mochi.
- Monitor rearrangement cannot silently place Mochi off-screen.

## Phase 4: Global Shortcuts portal

### Goal

Provide configurable Plasma-native shortcuts without depending on private KWin APIs.

### Backend priority

1. XDG Desktop Portal `GlobalShortcuts`
2. KWin script fallback for deliberately supported older Plasma versions
3. Disabled shortcut provider with a clear diagnostic

### Shortcut identities

```text
mochi.emote-catalogue
mochi.developer-menu
```

Suggested defaults:

- `Ctrl+Alt+E` for the emote catalogue
- `Ctrl+Alt+Shift+M` for the developer menu

Users should be able to change these in Plasma System Settings.

### Portal lifecycle

Implement:

- Portal feature and version detection
- Session creation
- Signal subscription before binding
- One bind operation per session
- `Activated` handling
- `Deactivated` handling
- `ShortcutsChanged` handling
- Portal restart recovery
- Session close during shutdown
- Duplicate activation prevention
- New session creation only when the requested shortcut set changes

### Files

- `src/mochi/developer_shortcut.py`
- `src/mochi/emote_shortcut.py`
- Proposed `src/mochi/shortcuts.py`
- Proposed `src/mochi/global_shortcuts_portal.py`
- New portal contract and lifecycle tests

### Exit criteria

- Both shortcuts work after reboot.
- Plasma users can rebind shortcuts.
- Mochi or portal restarts do not duplicate activation.
- GNOME shortcut behavior continues to work.

## Phase 5: Shared activity and system backends

### Idle and session awareness

Extend `src/mochi/presence/session.py` to observe:

- `IdleHint`
- `IdleSinceHintMonotonic`
- Existing `LockedHint`
- Existing `Active`
- `PrepareForSleep`

Preserve Mochi's current 120-second behavior threshold rather than blindly adopting the system policy threshold.

GNOME may continue to supply higher-quality compositor activity, but basic idle and sleep should not disappear when the GNOME helper is unavailable.

### Typing

KDE fallback order:

1. A supported, privacy-safe KDE provider if one is proven during the feasibility spike
2. AT-SPI caret and text activity
3. No typing animation

The public KWin scripting API provides active-window information and global shortcut registration but does not document a general anonymous keyboard-event API. Do not add an undocumented input hook, keylogger, or private D-Bus interface merely to claim parity.

The feasibility spike must test:

- AT-SPI coverage in Kate, Konsole, Dolphin, Firefox, Chromium, and JetBrains applications
- Wayland accessibility activation
- False positives from password fields and text replacement
- Remote desktop input
- Virtual keyboards

If exact GNOME-equivalent keyboard pulses cannot be obtained through supported APIs, document reduced KDE typing coverage honestly.

### Battery

Improve UPower handling instead of introducing PowerDevil:

- Enumerate UPower devices
- Select the primary battery deterministically
- Handle systems without `/DisplayDevice`
- Preserve charging and low-battery behavior
- Fail closed if UPower is unavailable

### Network

Keep NetworkManager as the primary provider and distinguish:

- Unavailable
- Locally connected
- Internet validated

Do not depend on `plasma-nm` internals.

### Media and music

Keep MPRIS. A later improvement may replace one-second enumeration with:

- `NameOwnerChanged`
- `PropertiesChanged`
- Conservative metadata classification
- Player-disappearance recovery

### File browsing

Retain both:

- Active-window classification from the KDE provider
- Generic Downloads-directory monitoring

### Files

- `src/mochi/presence/session.py`
- `src/mochi/typing_activity.py`
- `src/mochi/presence_activity.py`
- `src/mochi/presence/signals.py`
- `src/mochi/file_activity.py`
- `src/mochi/media_activity.py`

### Exit criteria

- Idle, active, lock, suspend, resume, battery, network, MPRIS, and Downloads work without the GNOME extension.
- Every unavailable service degrades safely.
- No polling source or D-Bus subscription survives shutdown.

## Phase 6: KWin semantic activity provider

### Goal

Restore active-window AmbiSense parity on KDE.

### Components

Create a KWin script with:

- `metadata.json`
- `contents/code/main.js`
- Independent script version
- Dedicated Mochi D-Bus service
- Corresponding Python D-Bus consumer

The provider should emit:

- Coarse application category
- File-manager focused and stopped states
- Privacy-reduced focused-video or YouTube boolean
- Initial state snapshot
- Clean state when disabled
- Recovery after KWin or script reload

### Application classification

Use documented active-window identity such as:

- `desktopFileName`
- `resourceClass`
- Other stable non-sensitive identity fields

Emit only:

```text
vscode
editor
terminal
browser
media
pixel_art
unknown
```

Do not transmit complete desktop file names, class strings, or window titles.

### Privacy

For focused YouTube detection:

- Inspect a title only transiently inside the KWin script
- Immediately reduce it to a boolean
- Never send or log the title
- Do not collect documents, URLs, clipboard data, key values, or file paths

This preserves the privacy boundary described in `docs/ambisense.md`.

### Installation

For Plasma 6, use the documented KWin script package structure and `kpackagetool6`.

- A user installer may install and offer to enable the script
- A system package should install without silently editing `kwinrc`
- Enabling must use supported KWin configuration
- Uninstall must offer complete removal
- Disabling the script must not stop core Mochi

For Plasma 5, create a separate KF5 script variant only if that version remains in the support policy.

### Files

- Proposed `kwin-scripts/mochi-activity/metadata.json`
- Proposed `kwin-scripts/mochi-activity/contents/code/main.js`
- Proposed `src/mochi/kwin_activity.py`
- Proposed `tests/test_kwin_activity.py`
- Proposed private-bus integration test

### Exit criteria

- Dolphin, Konsole, Kate, browsers, and supported media applications classify correctly.
- Provider failure does not affect core Mochi.
- No sensitive window metadata crosses D-Bus.
- The KWin script is independently versioned and reversible.

## Phase 7: Desktop-aware installation and packaging

### Common installer

The common installation path should handle:

- Python virtual environment
- Application package
- Launcher
- Desktop entry
- Icon
- Generic startup configuration

### GNOME path

Only in GNOME sessions:

- Install GNOME dependencies
- Install and enable the GNOME extension
- Compile GNOME schemas
- Run the existing one-time enablement flow

### KDE path

Only in KDE Plasma sessions:

- Check GTK4
- Check `gtk4-layer-shell`
- Check KWin and Plasma capability
- Check `xdg-desktop-portal-kde`
- Install the KWin script
- Offer to enable full AmbiSense
- Do not install GNOME Shell or GNOME extension packages
- Do not require XWayland

### XWayland

Treat X11 and XWayland packages as optional for native Plasma. Offer an explicit compatibility option rather than installing XWayland unconditionally.

### Package targets

Recommended order:

1. Fedora KDE RPM
2. Arch package or AUR metadata
3. openSUSE package
4. Flatpak manifest
5. Nix package, if desired

### Flatpak considerations

A Flatpak needs:

- Session D-Bus
- Global Shortcuts portal
- Downloads access
- System-bus permission for logind
- System-bus permission for UPower
- System-bus permission for NetworkManager

A Flatpak cannot depend directly on an installed KWin script, so active-window AmbiSense will be reduced unless a future portal provides that capability.

### Files

- `install.sh`
- `uninstall.sh`
- `scripts/install-typing-extension.sh`
- `scripts/enable-gnome-helper-once.sh`
- `packaging/*`
- `pyproject.toml`
- `tests/test_installer.py`

### Exit criteria

- Clean KDE installation requires no GNOME packages or commands.
- Clean GNOME installation retains its extension workflow.
- Update and uninstall preserve user configuration predictably.
- Native KDE installation does not require XWayland.

## 8. Testing strategy

## 8.1 Fast PR tests

Existing baseline:

```bash
python3 -m pytest -q
python3 -m compileall -q src tests
git diff --check
python3 -m pip wheel . --no-deps --no-build-isolation -w /tmp/mochi-wheel
```

Add coverage for:

- KDE display selection
- Layer-shell output selection
- Placement migration
- Global Shortcuts portal lifecycle
- KWin provider lifecycle
- Installer desktop branching
- Privacy contracts
- KWin script metadata
- KWin JavaScript with a mocked KWin API

## 8.2 D-Bus integration tests

Use a private `dbus-run-session` and private service names to test:

- Provider absent at startup
- Provider appears later
- Provider restarts
- D-Bus owner changes
- State cleared on provider loss
- No stale state after reconnection
- Portal session close
- Duplicate activation prevention
- No real user shortcut changes

## 8.3 Headless Wayland tests

Run a nested Wayland environment for:

- Launch and mapping
- Transparency
- Layer-shell initialization
- Input
- Output creation and removal
- Scale changes
- Popup placement

A wlroots-based compositor validates protocol behavior but does not replace KWin testing.

## 8.4 Plasma VM tests

Run a real Plasma Wayland session covering:

- Oldest supported Plasma 6.x
- Current stable Plasma
- New beta as a canary
- GNOME Wayland and XWayland regression
- Plasma X11 regression

## 8.5 Manual release matrix

Every supported tier must test:

- Single monitor
- Two monitors with negative origin
- Mixed scaling
- Monitor hotplug
- Rotation
- Fullscreen applications
- Activities
- Virtual desktops
- Lock and unlock
- Suspend and resume
- Dolphin
- Konsole
- Kate
- Firefox and Chromium
- VLC or mpv
- PipeWire
- UPower battery simulation
- NetworkManager disconnect and reconnect
- Portal shortcut configuration and reboot persistence
- KWin script reload and removal
- GNOME regression

## 9. Privacy and security requirements

The KDE implementation must preserve these invariants:

- No typed text
- No key codes or key values
- No clipboard content
- No file paths
- No URLs
- No raw window titles in D-Bus or logs
- No complete application identities in shared semantic events
- No unapproved PowerDevil or private KWin input APIs
- No private GNOME configuration access on KDE
- No silent modification of `kwinrc` from a system package
- No real user shortcut changes in test suites
- No desktop-event collection beyond the documented semantic needs

## 10. Release milestones

| Milestone | Scope | Approximate effort |
|---|---|---:|
| M0 | Contracts, diagnostics, GNOME baseline | 1-2 weeks |
| M1 | Native Wayland backend and output handling | 2-3 weeks |
| M2 | Popups and placement migration | 2-3 weeks |
| M3 | Portal shortcuts and shared integrations | 2-4 weeks |
| M4 | KWin semantic provider | 3-5 weeks |
| M5 | KDE installer and first package target | 2-3 weeks |
| M6 | Plasma VM CI and release qualification | 2-4 weeks |

A single experienced developer should expect approximately 14-20 weeks for a strong first Plasma release. A Tier 1 core release can be reached earlier if active-window AmbiSense is introduced as an optional KWin component first.

## 11. Definition of done

A KDE Wayland release is fully supported only when:

- Mochi launches natively without forcing XWayland.
- No GNOME package, extension, or command is required.
- Layer-shell placement works across supported Plasma versions.
- Monitor removal and mixed-DPI layouts recover safely.
- Context menu, catalogue, Focus, bubbles, and overlays work correctly.
- Global shortcuts work and are configurable in Plasma System Settings.
- Idle, sleep, lock, resume, network, battery, media, and Downloads work.
- KWin AmbiSense meets the documented feature contract or is explicitly marked reduced.
- Installing, updating, and uninstalling do not damage GNOME state.
- GNOME Wayland and XWayland remain tested targets.
- The privacy contract is verified.
- Oldest-supported and current-release Plasma tests pass.
- Beta releases can run as canaries without affecting stable support policy.
- Documentation accurately lists every available and degraded capability.

## 12. Key decisions

1. Keep the GTK4 application instead of rewriting it in Qt.
2. Keep the state, animation, care, and Focus core desktop-neutral.
3. Use capability detection rather than Plasma-version branching.
4. Use GTK layer-shell as the native KDE Wayland window backend.
5. Use the XDG Global Shortcuts portal for Plasma shortcuts.
6. Use logind, UPower, NetworkManager, MPRIS, Downloads, and AT-SPI where appropriate.
7. Use an optional, privacy-safe KWin script for active-window context.
8. Do not depend on private KWin input APIs or PowerDevil internals.
9. Preserve GNOME as a first-class regression target.
10. Do not claim full feature parity where KDE exposes no supported equivalent; define and test reduced behavior instead.
