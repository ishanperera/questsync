# QuestSync Roadmap

This document tracks the development phases for QuestSync. Each phase delivers independently testable functionality.

## Phase 1: Foundation (In Progress)

**Goal:** Detect a Quest 3 over USB, implement the ADB protocol, achieve first authenticated connection.

- [ ] Project scaffolding (Swift Package, Xcode project)
- [ ] ADB message encoding/decoding (wire format)
- [ ] ADB transport abstraction (USB + mock for testing)
- [ ] RSA key generation and ADB authentication
- [ ] ADB connection state machine (CNXN → AUTH → CNXN)
- [ ] IOKit USB device detection
- [ ] Device Manager with connect/disconnect notifications
- [ ] Debug harness app (minimal SwiftUI)
- [ ] Hardware integration test with Quest 3

**Milestone:** "Hello Quest" — app detects device, authenticates, prints device info.

## Phase 2: File Operations

**Goal:** Browse the Quest filesystem and transfer files programmatically.

- [ ] ADB Sync service: LIST, STAT, RECV, SEND
- [ ] ADB Shell service for commands
- [ ] Read Quest directories into Swift models
- [ ] Push/pull single files via ADB sync
- [ ] Size-based integrity verification
- [ ] Optional SHA-256 verification via shell

**Milestone:** Browse filesystem and transfer files via tests/CLI.

## Phase 3: Transfer Engine

**Goal:** Reliable, queued transfers with progress tracking for multi-GB files.

- [ ] Transfer queue with pause/resume/cancel
- [ ] Chunked streaming with progress reporting
- [ ] Retry logic with exponential backoff
- [ ] Batch transfers (multiple files)
- [ ] Folder transfers (recursive)
- [ ] Queue persistence to disk (JSON)

**Milestone:** Reliable multi-GB transfers with real-time progress.

## Phase 4: Basic GUI

**Goal:** Usable three-panel file manager with drag-and-drop.

- [ ] Three-panel SwiftUI layout (sidebar, browser, queue)
- [ ] Breadcrumb navigation
- [ ] File list with sorting and multi-select
- [ ] Toolbar buttons (Download, Upload, Delete, New Folder, Rename)
- [ ] Drag-and-drop from Finder into QuestSync (upload)
- [ ] Transfer queue panel with real-time progress bars

**Milestone:** First usable GUI — browse and transfer files visually.

## Phase 5: Onboarding & Polish

**Goal:** Non-technical users can set up and use QuestSync without external help.

- [ ] First-launch welcome flow
- [ ] Developer Mode setup wizard (step-by-step with links)
- [ ] Connection guide ("approve on headset" screen)
- [ ] Contextual hint banners in file browser
- [ ] macOS notifications for transfer events
- [ ] Settings window (download location, verification, appearance)

**Milestone:** Download-to-first-transfer in under 5 minutes with guided setup.

## Phase 6: Power Features

**Goal:** Feature-complete v1.0 for power users and sideloaders.

- [ ] APK installation (one-click sideload)
- [ ] Right-click context menus
- [ ] Bookmarks (pin Quest directories)
- [ ] File type icons and image thumbnails
- [ ] Drag from QuestSync to Finder (download)
- [ ] Menu bar icon with transfer status
- [ ] Keyboard shortcuts

**Milestone:** Feature-complete v1.0.

## Phase 7: Release Prep

**Goal:** Public release with documentation, CI/CD, and distribution.

- [ ] README with screenshots and installation guide
- [ ] ARCHITECTURE.md (system design for contributors)
- [ ] ADB-PROTOCOL.md (protocol implementation reference)
- [ ] CONTRIBUTING.md with good-first-issue templates
- [ ] GitHub Actions CI (build + test on every PR)
- [ ] Automated DMG creation and Apple notarization
- [ ] Homebrew Cask formula
- [ ] GitHub issue/PR templates

**Milestone:** Public v1.0 release on GitHub.

## Post-v1.0

Features planned for future releases:

- **Wireless ADB** — Transfer files over Wi-Fi using ADB's TCP transport
- **Quest companion app** — Lightweight Android app enabling wireless transfers without Developer Mode
- **Video/3D model preview** — Preview VR content before transferring
- **Storage visualization** — See what's using space on your Quest
- **iOS app** — Wi-Fi-only companion app for iPhone/iPad
- **CLI tool** — `questsync push/pull` for scripting and automation
- **Mac App Store** — Sandboxed version (Wi-Fi only) for broader reach
