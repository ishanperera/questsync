# QuestSync — Design Specification

**Date:** 2026-03-14
**Status:** Approved
**License:** MIT

## Overview

QuestSync is a free, open-source macOS application for managing, browsing, and transferring files to and from Meta Quest 3 VR headsets. It replaces buggy alternatives (Android File Transfer) and paid closed-source tools with a polished, native macOS experience.

## Target Users

Progressive disclosure UX serving two personas:

- **Casual Quest owners** who want to move videos/screenshots with minimal setup
- **Power users / sideloaders** who install APKs, mods, and need full filesystem access

The UI starts simple and reveals advanced features as users explore.

## Requirements Summary

| Area | Decision |
|------|----------|
| Connection | USB and Wi-Fi as equal first-class options |
| iOS port | Future — don't over-engineer for it now |
| Quest companion app | Deferred — v1.0 is ADB-only (Developer Mode required) |
| File operations | Full file manager + APK install, batch ops, transfer queue, previews |
| Distribution | Direct (DMG/Homebrew) first, App Store later (Wi-Fi-only sandboxed) |
| ADB strategy | Native Swift ADB client (no external binary dependency) |
| UI layout | Three-panel with toolbar buttons + contextual hints |
| Cross-platform | macOS-first; architecture enables community ports later |
| License | MIT |
| Minimum macOS | 13 (Ventura) |

**Priorities (ordered):**

1. Transfer speed and reliability — large files must transfer without corruption
2. Easy setup — minimal steps from download to first transfer
3. Developer Mode onboarding — guided walkthrough for non-technical users
4. Beautiful native UI — polished macOS look and feel

## System Architecture

Two-layer architecture: a reusable core library and the GUI application.

```
┌─────────────────────────────────────────────────┐
│                 QuestSync.app                    │
│              (SwiftUI macOS App)                 │
│                                                  │
│  ┌───────────┐  ┌───────────┐  ┌──────────────┐ │
│  │  Sidebar   │  │   File    │  │  Transfer    │ │
│  │  (Devices, │  │  Browser  │  │   Queue      │ │
│  │ Bookmarks) │  │  (Center) │  │  (Right)     │ │
│  └───────────┘  └───────────┘  └──────────────┘ │
│                                                  │
│  ┌──────────────────────────────────────────────┐│
│  │         Onboarding / Setup Wizard            ││
│  └──────────────────────────────────────────────┘│
└──────────────────┬──────────────────────────────┘
                   │ Swift API
┌──────────────────▼──────────────────────────────┐
│              QuestSyncKit                        │
│         (Swift Package Library)                  │
│                                                  │
│  ┌──────────┐  ┌──────────┐  ┌───────────────┐  │
│  │   ADB    │  │ Transfer │  │    Device      │  │
│  │ Protocol │  │  Engine  │  │   Manager      │  │
│  │          │  │          │  │                 │  │
│  │ - Auth   │  │ - Queue  │  │ - Detection    │  │
│  │ - Shell  │  │ - Resume │  │ - USB (IOKit)  │  │
│  │ - Sync   │  │ - Verify │  │ - WiFi (NWF)  │  │
│  │ - Push   │  │ - Stream │  │ - Status       │  │
│  │ - Pull   │  │ - Batch  │  │                 │  │
│  └──────────┘  └──────────┘  └───────────────┘  │
└─────────────────────────────────────────────────┘
```

**QuestSyncKit** is a standalone Swift Package with zero UI dependencies. It can be consumed by the GUI app, a future CLI tool, or third-party developers.

## ADB Protocol Implementation

### Layer 1 — Transport (USB)

- IOKit for USB device detection using Meta's vendor ID (`0x2833`)
- Claim ADB USB interface (class `0xFF`, subclass `0x42`, protocol `0x01`)
- Read/write ADB messages over bulk USB endpoints
- Handle macOS quirks: kernel driver detachment, device sleep/wake reconnection
- Transport is abstracted behind an `ADBTransport` protocol for future TCP (Wi-Fi) support

### Layer 2 — ADB Wire Protocol

- **Message format:** 24-byte header (command, arg0, arg1, data length, checksum, magic) + variable data payload
- **Commands:** `CNXN` (connect), `AUTH` (RSA authentication), `OPEN` (open stream), `WRTE` (write data), `CLSE` (close stream), `OKAY` (acknowledgment)
- **Authentication:** RSA 2048 keypair generated on first connection. The private key is standard PKCS#8, but the public key sent to the device during `AUTH` must use Android's proprietary public key format (not standard PEM). Keypair stored in `~/.questsync/` with `0600` file permissions.

### Layer 3 — Services

- **Shell service** (`shell:` prefix) — Run commands: `ls`, `pm install`, `mkdir`, storage queries
- **Sync service** (`sync:` prefix) — File transfer protocol:
  - `LIST` — directory entries (name, mode, size, mtime)
  - `RECV` — pull file from Quest (chunked streaming)
  - `SEND` — push file to Quest (chunked streaming)
  - `STAT` — file metadata

### Wireless ADB (future)

Same ADB protocol over a TCP connection. On Android 11+ (which Quest 3 runs), wireless ADB uses a pairing-based flow: `adb pair` on a random port (requires a 6-digit code displayed on-device), followed by `adb connect` on a separate port. The legacy `adb tcpip 5555` method requires an initial USB connection to enable. The `ADBTransport` protocol abstraction means the protocol and service layers remain unchanged — only a new `TCPTransport` implementation is needed, plus a pairing flow in the UI.

**Note:** The Mac App Store sandboxed version depends on wireless ADB being implemented first — these are not independent items.

## Device Manager & Connection UX

### USB Detection

- IOKit notification for USB device matching on Meta vendor ID
- Quest 3 identified by product ID
- Connect/disconnect events published via Combine
- Handles macOS MTP mount interference: use `IOServiceAddMatchingNotification` to detect the device, then send a kernel driver detach request (`IOUSBDeviceInterface::USBDeviceReEnumerate` or `IORegistryEntrySetCFProperty` to request re-enumeration) before claiming the ADB interface. The macOS `usbd` and MTP daemon may claim the device first, so the app must be prepared to retry interface acquisition after detaching the kernel driver.

### Connection State Machine

```
No Device ──[USB detected]──► Connected (Unauthenticated)
    ▲                              │
    │                         Send AUTH
    │                              ▼
    └──[USB removed]────────  Awaiting Quest Approval
                                   │
                              User taps "Allow"
                                   ▼
                                 Ready
```

### First-Time Connection Flow

1. App opens → "No device detected" with friendly illustration
2. User plugs in Quest via USB-C
3. App detects device, initiates ADB handshake
4. First connection: app shows "Check your headset — put it on and tap 'Always Allow'" with visual guide
5. User approves → file browser opens
6. RSA key saved — future connections are instant

### Developer Mode Onboarding Wizard

Triggered when a Meta USB device is detected (by vendor ID `0x2833`) but the ADB-specific interface (class `0xFF`, subclass `0x42`, protocol `0x01`) is not present among the device's interfaces. This requires a broader IOKit match on vendor ID, followed by interface enumeration to check for ADB capability. If only MTP interfaces are found, Developer Mode is likely off:

1. "Developer Mode is required" — explains what/why in non-scary language
2. "Create a Meta Developer Account" — links to portal, explains it's free
3. "Enable Developer Mode" — step-by-step: Meta Horizon app → Devices → Quest → Developer Mode toggle
4. "Restart your headset"
5. "Plug in again" — app auto-detects and completes

Wizard is dismissable and accessible from Settings. Each step has "I've done this" to advance.

## Transfer Engine

### Queue Architecture

- `TransferTask` model: source path, destination path, direction (push/pull), total bytes, transferred bytes, status (queued/active/paused/completed/failed), retry count
- Single concurrent transfer (ADB sync is single-threaded per connection), architecture supports multiple for future Wi-Fi
- Queue persists to disk — survives app restarts

### Chunked Streaming

- Chunk size determined by the `maxdata` field negotiated during the `CNXN` handshake (typically 64KB on modern devices, but may be 4KB on older protocol versions) — never loads entire files into memory
- Progress published via Combine `@Published` properties for direct UI binding
- Rolling average speed (5-second window) for stable ETA estimates
- Per-file and aggregate queue progress

### Error Handling & Recovery

- USB disconnect → pause active transfers, notify user, auto-resume on reconnect
- Transfer timeout (30s no data) → retry up to 3 times with exponential backoff
- Disk full → detect via `STAT` before starting, fail with clear message
- Corrupted transfer → post-transfer size verification, optional SHA-256 via shell

### Batch Operations

- Multi-file drag-and-drop queues one `TransferTask` per file
- Folder transfers recursively enumerate and queue, preserving directory structure
- Cancel/pause per-item or entire queue

### APK Installation

- Push APK to temp location, run `pm install` via ADB shell
- Report success/failure with human-readable error messages
- Option to delete temp APK after successful install

## UI Design

### Three-Panel Layout

**Left Sidebar (200px, collapsible):**
- Devices with status indicator (green/yellow/red dot)
- Bookmarks (user-pinned Quest directories)
- Quick Actions: "Install APK", "Screenshots"

**Center File Browser (flexible width):**
- Breadcrumb navigation (clickable path segments)
- Toolbar: Download to Mac, Upload to Quest, New Folder, Delete, Rename
- Contextual hint banner for new users (dismissable, remembers dismissal)
- File list: icon, name, size, date modified — sortable columns
- Multi-select: Cmd+Click, Shift+Click
- Right-click context menu: Download, Upload, Rename, Delete, Copy Path, Get Info
- Drag from Finder into browser (upload), drag from browser to Finder (download)
- Empty state with illustration

**Right Transfer Queue (250px, collapsible):**
- Active: filename, progress bar, speed, ETA, pause/cancel
- Queued: listed with cancel buttons
- Completed: checkmark, auto-clear after 5 minutes
- Failed: retry button with error description
- Bottom: aggregate progress, Pause All, Clear Completed

### Additional UX

- **Menu bar icon** — transfer progress when minimized
- **Drag-and-drop overlay** — full-window "Drop to upload/download" on drag
- **macOS notifications** — transfer complete, failed, device events
- **Settings** — default download location, verification toggle, appearance, timeouts

### Onboarding

1. Welcome screen
2. Developer Mode wizard (if needed)
3. "Plug in your Quest" prompt
4. Connection approval guide
5. File browser — done

## Project Structure

```
questsync/
├── QuestSyncKit/                 # Swift Package (core library)
│   ├── Package.swift
│   └── Sources/
│       ├── ADB/
│       │   ├── ADBConnection.swift
│       │   ├── ADBMessage.swift
│       │   ├── ADBAuth.swift
│       │   ├── ADBShell.swift
│       │   └── ADBSync.swift
│       ├── Transport/
│       │   ├── ADBTransport.swift        # Protocol (interface)
│       │   ├── USBTransport.swift        # IOKit implementation
│       │   └── TCPTransport.swift        # Wi-Fi (future, stubbed)
│       ├── Device/
│       │   ├── DeviceManager.swift
│       │   └── QuestDevice.swift
│       └── Transfer/
│           ├── TransferEngine.swift
│           ├── TransferTask.swift
│           └── TransferVerifier.swift
├── QuestSync/                    # macOS SwiftUI App
│   ├── QuestSync.xcodeproj
│   ├── App/
│   │   ├── QuestSyncApp.swift
│   │   └── AppState.swift
│   ├── Views/
│   │   ├── MainWindow/
│   │   │   ├── SidebarView.swift
│   │   │   ├── FileBrowserView.swift
│   │   │   ├── TransferQueueView.swift
│   │   │   └── ToolbarView.swift
│   │   ├── Onboarding/
│   │   │   ├── WelcomeView.swift
│   │   │   ├── DeveloperModeWizard.swift
│   │   │   └── ConnectionGuideView.swift
│   │   └── Settings/
│   │       └── SettingsView.swift
│   ├── ViewModels/
│   │   ├── FileBrowserViewModel.swift
│   │   ├── TransferQueueViewModel.swift
│   │   └── DeviceViewModel.swift
│   ├── Assets.xcassets/
│   └── Info.plist
├── docs/
│   ├── ARCHITECTURE.md
│   ├── ADB-PROTOCOL.md
│   └── DEVELOPER-MODE-GUIDE.md
├── .github/
│   ├── ISSUE_TEMPLATE/
│   │   ├── bug_report.md
│   │   └── feature_request.md
│   └── workflows/
│       └── build.yml
├── CONTRIBUTING.md
├── ROADMAP.md
├── LICENSE
├── README.md
└── Makefile
```

## Distribution

### Primary: GitHub Releases (DMG)

- Signed `.app` bundle with Developer ID certificate
- Packaged as `.dmg` with drag-to-Applications
- Notarized with Apple (no Gatekeeper warnings)

### Secondary: Homebrew Cask

- `brew install --cask questsync`
- Points to GitHub Release DMG

### Future: Mac App Store

- Sandboxed version, Wi-Fi only (IOKit requires unsandboxed entitlements)
- Separate Xcode target with appropriate entitlements

### CI/CD (GitHub Actions)

- Build and test on every PR
- Automated DMG creation and notarization on tagged releases
- SwiftLint enforcement

## Development Roadmap

### Phase 1 — Foundation (Weeks 1-3)

- Repo setup, Swift Package, Xcode project
- ADB message encoding/decoding
- IOKit USB device detection
- ADB `CNXN` handshake
- RSA key generation and authentication
- **Milestone: "Hello Quest" — detect, authenticate, print device info**

### Phase 2 — File Operations (Weeks 4-7)

- ADB Sync: `LIST`, `STAT`, `RECV`, `SEND`
- ADB Shell service
- Read Quest directories into Swift models
- Push/pull single files
- Size-based integrity verification
- **Milestone: Browse filesystem and transfer files via tests/CLI**

### Phase 3 — Transfer Engine (Weeks 8-10)

- Transfer queue with pause/resume/cancel
- Chunked streaming with progress
- Retry logic and error recovery
- Batch and folder transfers
- Queue persistence to disk
- **Milestone: Reliable multi-GB transfers with progress**

### Phase 4 — Basic GUI (Weeks 11-14)

- Three-panel SwiftUI layout
- Breadcrumb navigation
- File list with sorting and multi-select
- Toolbar action buttons
- Drag-and-drop from Finder
- Transfer queue panel with real-time progress
- **Milestone: Usable GUI for browsing and transferring**

### Phase 5 — Onboarding & Polish (Weeks 15-17)

- Welcome flow
- Developer Mode wizard
- Connection guide
- Contextual hints
- macOS notifications
- Settings window
- **Milestone: Non-technical user can set up and transfer with guidance**

### Phase 6 — Power Features (Weeks 18-22)

- APK installation
- Context menus
- Bookmarks
- File type icons and thumbnails
- Drag from app to Finder
- Menu bar icon
- Keyboard shortcuts
- **Milestone: Feature-complete v1.0**

### Phase 7 — Release Prep (Weeks 23-25)

- README with screenshots and installation guide
- ARCHITECTURE.md and ADB-PROTOCOL.md
- CONTRIBUTING.md with good-first-issue templates
- ROADMAP.md
- GitHub Actions CI
- DMG packaging and notarization
- Homebrew Cask formula
- **Milestone: Public v1.0 release**

### Post-v1.0

- Wireless ADB (TCP transport)
- Quest companion app (no Developer Mode needed)
- Video/3D model preview
- Storage usage visualization
- iOS port (Wi-Fi only)
- CLI tool using QuestSyncKit

## Security

### Key Storage

- RSA keypair stored at `~/.questsync/adbkey` (private) and `~/.questsync/adbkey.pub` (public)
- File permissions set to `0600` (owner read/write only) on creation
- macOS Keychain is not used — the keypair needs to be file-accessible for the ADB protocol format and to allow easy backup/migration. This matches how standard `adb` stores keys in `~/.android/`.
- If a malicious actor gains access to the private key, they can authorize ADB connections to any Quest that has previously approved that key. Mitigation: standard file permission enforcement, and users can revoke trusted computers from Quest Settings → Developer.

### Data Privacy

- QuestSync collects no telemetry, analytics, or user data
- All transfers occur directly between the Mac and Quest — no data passes through external servers
- This will be stated in the README and in a PRIVACY.md file

## Logging & Diagnostics

- Application logging via Apple's unified logging system (`os.log`) with the subsystem `com.questsync.app`
- Log categories: `device`, `adb`, `transfer`, `ui`
- Debug-level logging can be enabled from Settings for troubleshooting
- "Copy Debug Info" button in Settings that captures: app version, macOS version, Quest model/firmware, recent log entries, and connection state
- Logs never contain file contents — only metadata (filenames, sizes, paths, error messages)

## Testing Strategy

- **Unit tests (QuestSyncKit):** ADB message encoding/decoding, auth key format, sync protocol parsing, transfer queue logic. Use mock `ADBTransport` implementations to test without hardware.
- **Integration tests:** Require a real Quest connected via USB. Marked with a `@Tag("hardware")` annotation so they can be skipped in CI. Test full connection → auth → file transfer cycle.
- **UI tests:** Basic SwiftUI snapshot/interaction tests for critical flows (onboarding wizard, file browser, transfer queue).
- **Performance benchmark:** Transfer throughput test against a known file size, targeting within 80% of native `adb push/pull` throughput.

## Persistence

- **Transfer queue:** Serialized as JSON to `~/Library/Application Support/QuestSync/queue.json`. Only queued and paused items are persisted — active transfers restart from the beginning on app relaunch (ADB sync does not support seek/resume natively).
- **Settings:** `UserDefaults` via `@AppStorage` for simple preferences (download location, verification toggle, appearance).
- **Bookmarks:** JSON file at `~/Library/Application Support/QuestSync/bookmarks.json`.
- **RSA keys:** `~/.questsync/` directory (see Security section).

## Update Strategy

- Integrate the Sparkle framework for in-app update checks
- Sparkle checks the GitHub Releases appcast feed on launch (configurable frequency)
- Users are notified of updates with changelog and can install with one click
- Homebrew Cask users receive updates through `brew upgrade`

## Device Compatibility

- **Supported:** Meta Quest 3 (primary target, fully tested)
- **Expected to work:** Quest 2, Quest Pro, Quest 3S — all use the same USB vendor ID (`0x2833`) and ADB protocol. Not actively tested in v1.0 but architecturally identical.
- **Not supported:** Non-Meta Android devices (different vendor IDs, different filesystem layouts). QuestSync is purpose-built for Quest devices.
- The app matches on Meta's USB vendor ID. If multiple Quest models share the same vendor ID (they do), all are detected. Device model is identified via the `ro.product.model` property retrieved over ADB shell after connection.

## Known Risks

### Platform Risk

Meta controls the Quest firmware and could change ADB behavior, add authentication requirements, or restrict shell access in a future update. Mitigation: monitor Meta firmware changelogs, maintain a compatibility matrix, and communicate breaking changes to users promptly via GitHub Releases and in-app notifications.

### ADB Server Conflict

If the user has Android Studio or the `adb` command-line tool running, the standard ADB server (port 5037) claims the USB device. QuestSync's direct USB approach will conflict. Mitigation: on launch, detect if an ADB server is running and present the user with options — either let QuestSync kill the server, or instruct them to stop it manually. Display a clear error message, not a silent failure.

### Native ADB Implementation Risk

Building a full ADB client from scratch in Swift is the highest-risk technical component. ADB has undocumented quirks and version-specific edge cases. Mitigation: Phase 1 includes an ADB protocol spike. If the native implementation stalls beyond Phase 2, the fallback plan is to bundle the `adb` binary as a transitional measure while continuing native development in parallel.

### Trademark

"QuestSync" contains "Quest," which is a Meta trademark. For a free open-source tool this is unlikely to cause issues, but if the project gains significant visibility or targets the App Store, Meta could object. Monitor and be prepared to rename if necessary.

## Known Limitations (v1.0)

- No MTP fallback — ADB only, Developer Mode required
- No wireless transfers (USB only)
- No multi-device simultaneous connections
- No file content preview (video playback, 3D model viewer)
- No Quest companion app
- Single concurrent transfer (ADB protocol limitation per connection)
- Transfer resume after USB disconnect restarts the current file from the beginning
