# QuestSync

**Open-source macOS file manager for Meta Quest VR headsets.**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![macOS 13+](https://img.shields.io/badge/macOS-13%2B-brightgreen.svg)]()
[![Swift 5.9+](https://img.shields.io/badge/Swift-5.9%2B-orange.svg)]()

QuestSync is a free, native macOS application for browsing, managing, and transferring files to and from Meta Quest 3 VR headsets. No more buggy Android File Transfer. No more paid third-party apps. Just plug in and go.

---

## Why QuestSync?

If you own a Meta Quest on macOS, you've probably experienced:

- **Android File Transfer** crashing, freezing, or refusing to detect your headset
- **Paid apps** charging $10-20 for basic file transfer functionality
- **Command-line ADB** working but being unusable for most people
- **Multi-gigabyte VR files** (360 videos, game mods, sideloaded APKs) timing out or corrupting mid-transfer

QuestSync fixes all of this with a clean, native macOS app built from scratch.

## Features

### Current (In Development)

- Native ADB protocol implementation in pure Swift (no external `adb` binary required)
- USB device detection via IOKit
- RSA authentication with automatic key management
- Connection state machine with full handshake support

### Planned (v1.0)

- **Three-panel file manager** — sidebar, file browser, transfer queue
- **Drag-and-drop transfers** — drag files between your Mac and Quest
- **Batch operations** — multi-file transfers with queue management
- **APK installation** — one-click sideloading with status reporting
- **Large file support** — chunked streaming that handles multi-GB files without memory issues
- **Transfer queue** — pause, resume, cancel, retry with progress tracking
- **Developer Mode wizard** — step-by-step guided setup for non-technical users
- **Bookmarks** — pin frequently used Quest directories
- **macOS notifications** — know when transfers complete, even when minimized

### Future

- Wireless ADB transfers over Wi-Fi
- Quest companion app (no Developer Mode required)
- iOS companion app
- Video and 3D model previews
- CLI tool for scripting and automation

## Screenshots

> Coming soon — the app is in early development (Phase 1: Foundation).

## Installation

### Prerequisites

- **macOS 13 (Ventura) or later**
- **Meta Quest 3** (Quest 2, Quest Pro, and Quest 3S expected to work but untested)
- **Developer Mode enabled** on your Quest ([setup guide](#enabling-developer-mode))
- **USB-C cable** connecting your Mac to your Quest

### Download

Releases are not yet available. Once v1.0 is ready:

**Option 1: Direct Download**
Download the latest `.dmg` from [GitHub Releases](../../releases).

**Option 2: Homebrew**
```bash
brew install --cask questsync
```

### Building from Source

```bash
# Clone the repository
git clone https://github.com/yourusername/questsync.git
cd questsync

# Build the core library
make build

# Run tests
make test
```

To build the full macOS app, open the Xcode project:
1. Open `QuestSync/QuestSync.xcodeproj` in Xcode
2. Select your signing team in project settings
3. Build and Run (Cmd+R)

## Enabling Developer Mode

QuestSync communicates with your Quest using ADB (Android Debug Bridge), which requires Developer Mode. Here's how to enable it:

### Step 1: Create a Meta Developer Account
1. Go to the [Meta Developer Portal](https://developer.oculus.com/)
2. Log in with your Meta account
3. Accept the developer terms (it's free, no payment required)

### Step 2: Enable Developer Mode
1. Open the **Meta Horizon** app on your phone
2. Tap **Devices** and select your Quest headset
3. Tap **Headset Settings**
4. Find **Developer Mode** and toggle it **ON**

### Step 3: Restart Your Quest
1. Hold the power button on your Quest
2. Select **Restart**
3. Wait for it to boot up

### Step 4: Approve USB Debugging
1. Connect your Quest to your Mac with a USB-C cable
2. Put on your Quest headset
3. You'll see a prompt: **"Allow USB debugging?"**
4. Check **"Always allow from this computer"**
5. Tap **Allow**

That's it! QuestSync will now be able to communicate with your Quest.

## Architecture

QuestSync is built as two layers:

```
┌─────────────────────────────────────────┐
│           QuestSync.app                  │
│         (SwiftUI macOS App)              │
│                                          │
│   Sidebar │ File Browser │ Transfer Queue│
└─────────────────┬────────────────────────┘
                  │
┌─────────────────▼────────────────────────┐
│           QuestSyncKit                    │
│      (Swift Package Library)              │
│                                          │
│   ADB Protocol │ Transfer │ Device Mgr   │
│   - Auth        │ Engine   │ - USB (IOKit)│
│   - Shell       │ - Queue  │ - Detection  │
│   - Sync        │ - Stream │ - Status     │
└──────────────────────────────────────────┘
```

**QuestSyncKit** is a standalone Swift Package with zero UI dependencies. It implements the ADB protocol natively in Swift — no external `adb` binary, no shelling out, no dependencies. This means:

- **Zero setup** — no need to install Android Platform Tools
- **Full control** — we handle USB detection, authentication, and file transfer directly
- **Portable** — the library can be used by other apps, a CLI tool, or ported to other platforms

### Key Technical Decisions

| Decision | Rationale |
|----------|-----------|
| Native Swift ADB client | No external dependencies, best UX, full control over the protocol |
| IOKit for USB | Direct USB access on macOS, handles device detection and bulk I/O |
| SwiftUI | Modern macOS UI framework, less code, native look and feel |
| Swift Package Manager | Clean dependency management, easy for contributors to build |
| Transport protocol abstraction | USB today, TCP/Wi-Fi tomorrow — same protocol layer |

## Device Compatibility

| Device | Status |
|--------|--------|
| Meta Quest 3 | Supported (primary target) |
| Meta Quest 3S | Expected to work (untested) |
| Meta Quest Pro | Expected to work (untested) |
| Meta Quest 2 | Expected to work (untested) |
| Non-Meta Android | Not supported |

All Meta Quest devices share the same USB vendor ID (`0x2833`) and ADB protocol, so compatibility is expected across the lineup. If you test with a device not listed above, please [open an issue](../../issues) to let us know!

## Roadmap

See [ROADMAP.md](ROADMAP.md) for the full development plan.

| Phase | Focus | Status |
|-------|-------|--------|
| 1. Foundation | ADB protocol, USB detection, authentication | In Progress |
| 2. File Operations | Browse, push, pull files via ADB sync | Planned |
| 3. Transfer Engine | Queue, progress, retry, batch transfers | Planned |
| 4. Basic GUI | Three-panel file manager with drag-and-drop | Planned |
| 5. Onboarding | Developer Mode wizard, setup flow | Planned |
| 6. Power Features | APK install, bookmarks, thumbnails, menu bar | Planned |
| 7. Release | README, CI/CD, DMG packaging, Homebrew | Planned |

## Contributing

Contributions are welcome! QuestSync is a solo-led project, but PRs from the community are appreciated.

### Getting Started

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/my-feature`)
3. Build and test (`make build && make test`)
4. Commit your changes
5. Open a Pull Request

### Good First Issues

Look for issues labeled [`good first issue`](../../labels/good%20first%20issue) — these are scoped, well-documented tasks designed for new contributors.

### Development Setup

- **Xcode 15+** (for SwiftUI and macOS 13 SDK)
- **Swift 5.9+**
- **A Meta Quest headset** (for integration testing — unit tests work without hardware)

### Project Structure

```
questsync/
├── QuestSyncKit/           # Core library (Swift Package)
│   ├── Sources/
│   │   └── QuestSyncKit/
│   │       ├── ADB/        # ADB protocol implementation
│   │       ├── Transport/  # USB and network transport layers
│   │       └── Device/     # Device detection and management
│   └── Tests/
├── QuestSync/              # macOS SwiftUI application
├── docs/                   # Architecture and protocol documentation
└── Makefile                # Build shortcuts
```

## Privacy

QuestSync collects **no telemetry, analytics, or user data**. All file transfers occur directly between your Mac and your Quest over USB — nothing passes through external servers. Ever.

## License

MIT License — see [LICENSE](LICENSE) for details.

## Acknowledgments

- The [Android ADB protocol documentation](https://android.googlesource.com/platform/packages/modules/adb/+/refs/heads/main/protocol.txt) for making this possible
- The Meta Quest community for testing and feedback
- Everyone frustrated with Android File Transfer — you inspired this project
