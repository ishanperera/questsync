# Phase 1: Foundation — Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Establish the project structure, implement the ADB wire protocol in pure Swift, detect a Quest 3 over USB via IOKit, and achieve a successful authenticated "Hello Quest" handshake.

**Architecture:** QuestSyncKit is a Swift Package containing the ADB protocol implementation and USB transport layer. The transport is abstracted behind a protocol for future TCP support. ADB messages are 24-byte headers + data payloads. Authentication uses RSA 2048 with Android's proprietary public key format.

**Tech Stack:** Swift 5.9+, SwiftUI (minimal — just for a debug harness), IOKit (USB), Security.framework (RSA), Swift Package Manager, XCTest

**Spec:** `docs/superpowers/specs/2026-03-14-questsync-design.md`

---

## File Structure

```
questsync/
├── QuestSyncKit/
│   ├── Package.swift                          # Swift Package manifest
│   └── Sources/
│       └── QuestSyncKit/
│           ├── ADB/
│           │   ├── ADBMessage.swift           # Wire format: encode/decode 24-byte headers + data
│           │   ├── ADBConnection.swift        # Protocol state machine: CNXN → AUTH → OPEN → WRTE/CLSE
│           │   └── ADBAuth.swift              # RSA keypair generation, Android public key format, signing
│           ├── Transport/
│           │   ├── ADBTransport.swift         # Protocol (interface) for read/write/open/close
│           │   ├── USBTransport.swift         # IOKit USB implementation: detect, claim, bulk I/O
│           │   └── MockTransport.swift        # In-memory mock for unit tests
│           └── Device/
│               ├── DeviceManager.swift        # IOKit notifications, device connect/disconnect events
│               └── QuestDevice.swift          # Device model: serial, model, state, connection
├── QuestSyncKit/Tests/
│   └── QuestSyncKitTests/
│       ├── ADBMessageTests.swift              # Encode/decode round-trips, checksum, magic validation
│       ├── ADBAuthTests.swift                 # Key generation, Android format, signing
│       ├── ADBConnectionTests.swift           # State machine transitions using MockTransport
│       └── MockTransportTests.swift           # Verify mock behaves correctly
├── QuestSync/                                 # Minimal macOS app (debug harness for Phase 1)
│   ├── QuestSyncApp.swift                     # App entry point
│   └── ContentView.swift                      # Shows device status + connection log
├── LICENSE
├── README.md
├── .gitignore
└── Makefile
```

---

## Chunk 1: Project Scaffolding & ADB Message Format

### Task 1: Project Setup

**Files:**
- Create: `QuestSyncKit/Package.swift`
- Create: `QuestSyncKit/Sources/QuestSyncKit/ADB/ADBMessage.swift` (placeholder)
- Create: `LICENSE`
- Create: `README.md`
- Create: `.gitignore`
- Create: `Makefile`

- [ ] **Step 0: Initialize git repository**

```bash
cd /Users/ishanperera/Desktop/Projects/questprojects/questfiletransfer
git init
```

- [ ] **Step 1: Create the Swift Package**

```bash
cd /Users/ishanperera/Desktop/Projects/questprojects/questfiletransfer
mkdir -p QuestSyncKit/Sources/QuestSyncKit/ADB
mkdir -p QuestSyncKit/Sources/QuestSyncKit/Transport
mkdir -p QuestSyncKit/Sources/QuestSyncKit/Device
mkdir -p QuestSyncKit/Tests/QuestSyncKitTests
```

- [ ] **Step 2: Write Package.swift**

Create `QuestSyncKit/Package.swift`:

```swift
// swift-tools-version: 5.9
import PackageDescription

let package = Package(
    name: "QuestSyncKit",
    platforms: [
        .macOS(.v13)
    ],
    products: [
        .library(
            name: "QuestSyncKit",
            targets: ["QuestSyncKit"]
        )
    ],
    targets: [
        .target(
            name: "QuestSyncKit",
            path: "Sources/QuestSyncKit",
            linkerSettings: [
                .linkedFramework("IOKit"),
                .linkedFramework("Security")
            ]
        ),
        .testTarget(
            name: "QuestSyncKitTests",
            dependencies: ["QuestSyncKit"],
            path: "Tests/QuestSyncKitTests"
        )
    ]
)
```

- [ ] **Step 3: Create placeholder source file**

Create `QuestSyncKit/Sources/QuestSyncKit/ADB/ADBMessage.swift`:

```swift
import Foundation

// ADB Protocol Message - placeholder
public enum ADBCommand: UInt32 {
    case connect = 0x4e584e43    // "CNXN"
    case auth    = 0x48545541    // "AUTH"
    case open    = 0x4e45504f    // "OPEN"
    case okay   = 0x59414b4f    // "OKAY"
    case close  = 0x45534c43    // "CLSE"
    case write  = 0x45545257    // "WRTE"
}
```

- [ ] **Step 4: Create LICENSE (MIT)**

Create `LICENSE`:

```
MIT License

Copyright (c) 2026 QuestSync Contributors

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

- [ ] **Step 5: Create .gitignore**

Create `.gitignore`:

```
# Xcode
*.xcodeproj/xcuserdata/
*.xcworkspace/xcuserdata/
*.xcodeproj/project.xcworkspace/xcshareddata/IDEWorkspaceChecks.plist
DerivedData/
*.pbxuser
*.mode1v3
*.mode2v3
*.perspectivev3
*.moved-aside
build/

# Swift Package Manager
.build/
.swiftpm/
Package.resolved

# macOS
.DS_Store
*.swp
*~

# QuestSync
.superpowers/
```

- [ ] **Step 6: Create Makefile**

Create `Makefile`:

```makefile
.PHONY: build test clean

build:
	cd QuestSyncKit && swift build

test:
	cd QuestSyncKit && swift test

clean:
	cd QuestSyncKit && swift package clean
```

- [ ] **Step 7: Create initial README.md**

Create `README.md`:

```markdown
# QuestSync

Open-source macOS file manager for Meta Quest VR headsets.

> **Status:** Early development — Phase 1 (Foundation)

## What is QuestSync?

QuestSync is a free, open-source macOS application for managing, browsing, and transferring files to and from Meta Quest 3 VR headsets. It replaces buggy alternatives like Android File Transfer with a polished, native macOS experience.

## Building

```bash
make build
make test
```

## License

MIT — see [LICENSE](LICENSE) for details.
```

- [ ] **Step 8: Verify the package builds**

Run: `cd /Users/ishanperera/Desktop/Projects/questprojects/questfiletransfer/QuestSyncKit && swift build`
Expected: Build succeeds with no errors.

- [ ] **Step 9: Commit**

```bash
git add QuestSyncKit/Package.swift QuestSyncKit/Sources/ LICENSE README.md .gitignore Makefile
git commit -m "feat: scaffold QuestSyncKit Swift Package with project structure"
```

---

### Task 2: ADB Message Encoding/Decoding

**Files:**
- Create: `QuestSyncKit/Sources/QuestSyncKit/ADB/ADBMessage.swift` (replace placeholder)
- Create: `QuestSyncKit/Tests/QuestSyncKitTests/ADBMessageTests.swift`

The ADB wire protocol message format is:
- 24-byte header: command (4 bytes LE), arg0 (4 bytes LE), arg1 (4 bytes LE), data_length (4 bytes LE), data_checksum (4 bytes LE), magic (4 bytes LE)
- magic = command XOR 0xFFFFFFFF
- data_checksum = sum of all bytes in data payload (unsigned)
- Followed by `data_length` bytes of payload

- [ ] **Step 1: Write failing tests for ADBMessage**

Create `QuestSyncKit/Tests/QuestSyncKitTests/ADBMessageTests.swift`:

```swift
import XCTest
@testable import QuestSyncKit

final class ADBMessageTests: XCTestCase {

    func testCommandRawValues() {
        // ADB commands are ASCII strings stored as little-endian UInt32
        XCTAssertEqual(ADBCommand.connect.rawValue, 0x4e584e43)
        XCTAssertEqual(ADBCommand.auth.rawValue, 0x48545541)
        XCTAssertEqual(ADBCommand.open.rawValue, 0x4e45504f)
        XCTAssertEqual(ADBCommand.okay.rawValue, 0x59414b4f)
        XCTAssertEqual(ADBCommand.close.rawValue, 0x45534c43)
        XCTAssertEqual(ADBCommand.write.rawValue, 0x45545257)
    }

    func testMagicCalculation() {
        // Magic = command XOR 0xFFFFFFFF
        let message = ADBMessage(command: .connect, arg0: 0, arg1: 0, data: Data())
        XCTAssertEqual(message.magic, ADBCommand.connect.rawValue ^ 0xFFFFFFFF)
    }

    func testChecksumEmptyData() {
        let message = ADBMessage(command: .connect, arg0: 0, arg1: 0, data: Data())
        XCTAssertEqual(message.checksum, 0)
    }

    func testChecksumWithData() {
        // Checksum = sum of all bytes in data
        let data = Data([0x01, 0x02, 0x03, 0x04])
        let message = ADBMessage(command: .connect, arg0: 0, arg1: 0, data: data)
        XCTAssertEqual(message.checksum, UInt32(0x01 + 0x02 + 0x03 + 0x04))
    }

    func testSerializeHeaderSize() {
        let message = ADBMessage(command: .connect, arg0: 1, arg1: 256, data: Data())
        let serialized = message.serialize()
        // Header is 24 bytes, no data
        XCTAssertEqual(serialized.count, 24)
    }

    func testSerializeWithData() {
        let data = Data("host::features=shell_v2".utf8)
        let message = ADBMessage(command: .connect, arg0: 0x01000001, arg1: 4096, data: data)
        let serialized = message.serialize()
        // 24 byte header + data
        XCTAssertEqual(serialized.count, 24 + data.count)
    }

    func testSerializeDeserializeRoundTrip() throws {
        let data = Data("host::features=shell_v2,stat_v2".utf8)
        let original = ADBMessage(command: .connect, arg0: 0x01000001, arg1: 262144, data: data)
        let serialized = original.serialize()

        let deserialized = try ADBMessage.deserializeHeader(from: serialized)

        XCTAssertEqual(deserialized.command, original.command)
        XCTAssertEqual(deserialized.arg0, original.arg0)
        XCTAssertEqual(deserialized.arg1, original.arg1)
        XCTAssertEqual(deserialized.expectedDataLength, UInt32(data.count))
    }

    func testDeserializeInvalidMagic() {
        // Create valid header bytes, then corrupt the magic field
        var bytes = ADBMessage(command: .connect, arg0: 0, arg1: 0, data: Data()).serialize()
        // Magic is at offset 20-23, corrupt it
        bytes[20] = 0xFF
        bytes[21] = 0xFF
        bytes[22] = 0xFF
        bytes[23] = 0xFF

        XCTAssertThrowsError(try ADBMessage.deserializeHeader(from: bytes)) { error in
            XCTAssertTrue(error is ADBMessageError)
        }
    }

    func testDeserializeTooShort() {
        let bytes = Data([0x00, 0x01, 0x02]) // Only 3 bytes, need 24

        XCTAssertThrowsError(try ADBMessage.deserializeHeader(from: bytes)) { error in
            XCTAssertTrue(error is ADBMessageError)
        }
    }

    func testConnectMessageFormat() {
        // A CNXN message has: version in arg0, maxdata in arg1, system identity in data
        let identity = "host::\0"
        let message = ADBMessage.connect(version: 0x01000001, maxData: 262144, systemIdentity: identity)

        XCTAssertEqual(message.command, .connect)
        XCTAssertEqual(message.arg0, 0x01000001)
        XCTAssertEqual(message.arg1, 262144)
        XCTAssertEqual(String(data: message.data, encoding: .utf8), identity)
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd /Users/ishanperera/Desktop/Projects/questprojects/questfiletransfer/QuestSyncKit && swift test`
Expected: Compilation errors — `ADBMessage`, `ADBMessageError`, etc. not yet defined.

- [ ] **Step 3: Implement ADBMessage**

Replace `QuestSyncKit/Sources/QuestSyncKit/ADB/ADBMessage.swift`:

```swift
import Foundation

/// ADB protocol commands — ASCII strings stored as little-endian UInt32.
public enum ADBCommand: UInt32, Sendable {
    case connect = 0x4e584e43    // "CNXN"
    case auth    = 0x48545541    // "AUTH"
    case open    = 0x4e45504f    // "OPEN"
    case okay   = 0x59414b4f    // "OKAY"
    case close  = 0x45534c43    // "CLSE"
    case write  = 0x45545257    // "WRTE"
}

/// Errors during ADB message serialization/deserialization.
public enum ADBMessageError: Error, Sendable {
    case headerTooShort(got: Int, need: Int)
    case invalidMagic(expected: UInt32, got: UInt32)
    case unknownCommand(UInt32)
}

/// A single ADB protocol message (header + optional data payload).
///
/// Wire format (all little-endian):
/// ```
/// Offset  Size  Field
/// 0       4     command
/// 4       4     arg0
/// 8       4     arg1
/// 12      4     data_length
/// 16      4     data_checksum
/// 20      4     magic (command XOR 0xFFFFFFFF)
/// 24      N     data payload
/// ```
public struct ADBMessage: Sendable {
    public static let headerSize = 24

    public let command: ADBCommand
    public let arg0: UInt32
    public let arg1: UInt32
    public let data: Data
    /// When deserialized from a header, this holds the expected data length
    /// (since `data` is empty until the payload is read separately).
    public let expectedDataLength: UInt32?

    public init(command: ADBCommand, arg0: UInt32, arg1: UInt32, data: Data, expectedDataLength: UInt32? = nil) {
        self.command = command
        self.arg0 = arg0
        self.arg1 = arg1
        self.data = data
        self.expectedDataLength = expectedDataLength
    }

    /// Checksum: sum of all bytes in the data payload.
    public var checksum: UInt32 {
        data.reduce(UInt32(0)) { $0 + UInt32($1) }
    }

    /// Magic: command XOR 0xFFFFFFFF.
    public var magic: UInt32 {
        command.rawValue ^ 0xFFFFFFFF
    }

    /// Length of the data payload.
    public var dataLength: UInt32 {
        UInt32(data.count)
    }

    /// Serialize to bytes (header + data) for transmission.
    public func serialize() -> Data {
        var result = Data(capacity: Self.headerSize + data.count)
        result.appendLittleEndian(command.rawValue)
        result.appendLittleEndian(arg0)
        result.appendLittleEndian(arg1)
        result.appendLittleEndian(dataLength)
        result.appendLittleEndian(checksum)
        result.appendLittleEndian(magic)
        result.append(data)
        return result
    }

    /// Deserialize a 24-byte header from raw bytes.
    /// Returns a message with the header fields populated and `dataLength` indicating
    /// how many payload bytes to read next. The `data` field is empty — caller must
    /// read `dataLength` bytes separately and attach them.
    public static func deserializeHeader(from bytes: Data) throws -> ADBMessage {
        guard bytes.count >= headerSize else {
            throw ADBMessageError.headerTooShort(got: bytes.count, need: headerSize)
        }

        let commandRaw = bytes.readLittleEndianUInt32(at: 0)
        let arg0 = bytes.readLittleEndianUInt32(at: 4)
        let arg1 = bytes.readLittleEndianUInt32(at: 8)
        let dataLength = bytes.readLittleEndianUInt32(at: 12)
        // checksum at offset 16 — validated after data is read
        let magicRaw = bytes.readLittleEndianUInt32(at: 20)

        guard let command = ADBCommand(rawValue: commandRaw) else {
            throw ADBMessageError.unknownCommand(commandRaw)
        }

        let expectedMagic = commandRaw ^ 0xFFFFFFFF
        guard magicRaw == expectedMagic else {
            throw ADBMessageError.invalidMagic(expected: expectedMagic, got: magicRaw)
        }

        // Return message with empty data and expectedDataLength set.
        // Caller reads `expectedDataLength` bytes of payload separately.
        return ADBMessage(command: command, arg0: arg0, arg1: arg1, data: Data(), expectedDataLength: dataLength)
    }

    // MARK: - Factory Methods

    /// Create a CNXN (connect) message.
    public static func connect(version: UInt32, maxData: UInt32, systemIdentity: String) -> ADBMessage {
        ADBMessage(
            command: .connect,
            arg0: version,
            arg1: maxData,
            data: Data(systemIdentity.utf8)
        )
    }

    /// Create an AUTH message with a signature.
    public static func authSignature(_ signature: Data) -> ADBMessage {
        ADBMessage(command: .auth, arg0: 1, arg1: 0, data: signature)
    }

    /// Create an AUTH message with a public key.
    public static func authPublicKey(_ publicKey: Data) -> ADBMessage {
        ADBMessage(command: .auth, arg0: 3, arg1: 0, data: publicKey)
    }

    /// Create an OPEN message to open a stream.
    public static func open(localID: UInt32, destination: String) -> ADBMessage {
        ADBMessage(
            command: .open,
            arg0: localID,
            arg1: 0,
            data: Data((destination + "\0").utf8)
        )
    }

    /// Create an OKAY acknowledgment.
    public static func okay(localID: UInt32, remoteID: UInt32) -> ADBMessage {
        ADBMessage(command: .okay, arg0: localID, arg1: remoteID, data: Data())
    }

    /// Create a WRTE message with data payload.
    public static func write(localID: UInt32, remoteID: UInt32, data: Data) -> ADBMessage {
        ADBMessage(command: .write, arg0: localID, arg1: remoteID, data: data)
    }

    /// Create a CLSE message to close a stream.
    public static func close(localID: UInt32, remoteID: UInt32) -> ADBMessage {
        ADBMessage(command: .close, arg0: localID, arg1: remoteID, data: Data())
    }
}

// MARK: - Data Extensions for Little-Endian I/O

extension Data {
    mutating func appendLittleEndian(_ value: UInt32) {
        var le = value.littleEndian
        append(Data(bytes: &le, count: 4))
    }

    func readLittleEndianUInt32(at offset: Int) -> UInt32 {
        let i = self.startIndex + offset
        return UInt32(self[i])
             | UInt32(self[i + 1]) << 8
             | UInt32(self[i + 2]) << 16
             | UInt32(self[i + 3]) << 24
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd /Users/ishanperera/Desktop/Projects/questprojects/questfiletransfer/QuestSyncKit && swift test`
Expected: All tests in `ADBMessageTests` pass.

- [ ] **Step 5: Commit**

```bash
git add QuestSyncKit/Sources/QuestSyncKit/ADB/ADBMessage.swift QuestSyncKit/Tests/QuestSyncKitTests/ADBMessageTests.swift
git commit -m "feat: implement ADB message encoding/decoding with wire format"
```

---

### Task 3: ADB Transport Protocol & Mock

**Files:**
- Create: `QuestSyncKit/Sources/QuestSyncKit/Transport/ADBTransport.swift`
- Create: `QuestSyncKit/Sources/QuestSyncKit/Transport/MockTransport.swift`
- Create: `QuestSyncKit/Tests/QuestSyncKitTests/MockTransportTests.swift`

- [ ] **Step 1: Write failing tests for MockTransport**

Create `QuestSyncKit/Tests/QuestSyncKitTests/MockTransportTests.swift`:

```swift
import XCTest
@testable import QuestSyncKit

final class MockTransportTests: XCTestCase {

    func testSendAndReceive() async throws {
        let transport = MockTransport()
        try await transport.open()

        // Enqueue a response the mock will return on receive
        let response = ADBMessage(command: .connect, arg0: 0x01000001, arg1: 262144, data: Data("device::".utf8))
        transport.enqueueResponse(response)

        // Send a message (mock records it)
        let sent = ADBMessage.connect(version: 0x01000001, maxData: 262144, systemIdentity: "host::\0")
        try await transport.send(sent)

        // Receive the enqueued response
        let received = try await transport.receive()

        XCTAssertEqual(received.command, .connect)
        XCTAssertEqual(transport.sentMessages.count, 1)
        XCTAssertEqual(transport.sentMessages[0].command, .connect)
    }

    func testReceiveWithNoResponseThrows() async {
        let transport = MockTransport()

        do {
            _ = try await transport.receive()
            XCTFail("Expected error when no responses enqueued")
        } catch {
            XCTAssertTrue(error is ADBTransportError)
        }
    }

    func testOpenAndClose() async throws {
        let transport = MockTransport()

        XCTAssertFalse(transport.isOpen)
        try await transport.open()
        XCTAssertTrue(transport.isOpen)
        await transport.close()
        XCTAssertFalse(transport.isOpen)
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd /Users/ishanperera/Desktop/Projects/questprojects/questfiletransfer/QuestSyncKit && swift test`
Expected: Compilation errors — `ADBTransport`, `MockTransport`, `ADBTransportError` not defined.

- [ ] **Step 3: Implement ADBTransport protocol**

Create `QuestSyncKit/Sources/QuestSyncKit/Transport/ADBTransport.swift`:

```swift
import Foundation

/// Errors from the transport layer.
public enum ADBTransportError: Error, Sendable {
    case notConnected
    case connectionLost
    case timeout
    case noDataAvailable
}

/// Abstraction over the physical connection to an ADB device.
/// Implemented by USBTransport (IOKit) and MockTransport (tests).
public protocol ADBTransport: Sendable {
    /// Whether the transport connection is currently open.
    var isOpen: Bool { get }

    /// Open the transport connection.
    func open() async throws

    /// Close the transport connection.
    func close() async

    /// Send a complete ADB message (header + data).
    func send(_ message: ADBMessage) async throws

    /// Receive a complete ADB message (reads header, then data payload).
    func receive() async throws -> ADBMessage
}
```

- [ ] **Step 4: Implement MockTransport**

Create `QuestSyncKit/Sources/QuestSyncKit/Transport/MockTransport.swift`:

```swift
import Foundation

/// In-memory mock transport for unit testing ADB protocol logic without hardware.
public final class MockTransport: ADBTransport, @unchecked Sendable {
    private let lock = NSLock()
    private var _isOpen = false
    private var _sentMessages: [ADBMessage] = []
    private var _responseQueue: [ADBMessage] = []

    public var isOpen: Bool {
        lock.withLock { _isOpen }
    }

    /// All messages that were sent through this transport.
    public var sentMessages: [ADBMessage] {
        lock.withLock { _sentMessages }
    }

    public init() {}

    /// Enqueue a message that will be returned by the next `receive()` call.
    public func enqueueResponse(_ message: ADBMessage) {
        lock.withLock { _responseQueue.append(message) }
    }

    /// Enqueue multiple responses in order.
    public func enqueueResponses(_ messages: [ADBMessage]) {
        lock.withLock { _responseQueue.append(contentsOf: messages) }
    }

    public func open() async throws {
        lock.withLock { _isOpen = true }
    }

    public func close() async {
        lock.withLock { _isOpen = false }
    }

    public func send(_ message: ADBMessage) async throws {
        let open = lock.withLock { _isOpen }
        guard open else { throw ADBTransportError.notConnected }
        lock.withLock { _sentMessages.append(message) }
    }

    public func receive() async throws -> ADBMessage {
        let open = lock.withLock { _isOpen }
        guard open else { throw ADBTransportError.notConnected }

        return try lock.withLock {
            guard !_responseQueue.isEmpty else {
                throw ADBTransportError.noDataAvailable
            }
            return _responseQueue.removeFirst()
        }
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `cd /Users/ishanperera/Desktop/Projects/questprojects/questfiletransfer/QuestSyncKit && swift test`
Expected: All tests pass (ADBMessageTests + MockTransportTests).

- [ ] **Step 6: Commit**

```bash
git add QuestSyncKit/Sources/QuestSyncKit/Transport/ QuestSyncKit/Tests/QuestSyncKitTests/MockTransportTests.swift
git commit -m "feat: add ADBTransport protocol and MockTransport for testing"
```

---

### Task 4: ADB Authentication (RSA Key Generation)

**Files:**
- Create: `QuestSyncKit/Sources/QuestSyncKit/ADB/ADBAuth.swift`
- Create: `QuestSyncKit/Tests/QuestSyncKitTests/ADBAuthTests.swift`

ADB authentication flow:
1. Device sends `AUTH` with type=1 (TOKEN), containing a 20-byte random token
2. Host signs the token with its RSA private key, sends `AUTH` with type=2 (SIGNATURE)
3. If device recognizes the key → sends `CNXN` (success)
4. If not → host sends `AUTH` with type=3 (RSAPUBLICKEY), containing the public key in Android's format
5. Device shows "Allow USB debugging?" prompt to user
6. User approves → device sends `CNXN`

Android's public key format: the raw RSA public key as a struct (n_words, n0inv, n[], rr[], exponent), base64-encoded, followed by ` host@hostname\0`

- [ ] **Step 1: Write failing tests for ADBAuth**

Create `QuestSyncKit/Tests/QuestSyncKitTests/ADBAuthTests.swift`:

```swift
import XCTest
@testable import QuestSyncKit

final class ADBAuthTests: XCTestCase {

    func testGenerateKeyPair() throws {
        let auth = ADBAuth()
        let keyPair = try auth.generateKeyPair()

        // Private key should be non-empty DER data
        XCTAssertGreaterThan(keyPair.privateKeyData.count, 0)
        // Public key in Android format should be non-empty
        XCTAssertGreaterThan(keyPair.androidPublicKey.count, 0)
    }

    func testAndroidPublicKeyFormat() throws {
        let auth = ADBAuth()
        let keyPair = try auth.generateKeyPair()

        // Android public key should end with a user@host identifier and null terminator
        let pubKeyString = String(data: keyPair.androidPublicKey, encoding: .utf8)
        XCTAssertNotNil(pubKeyString)
        // Should contain base64 data followed by a space and identifier
        XCTAssertTrue(pubKeyString!.contains(" "))
    }

    func testSignToken() throws {
        let auth = ADBAuth()
        let keyPair = try auth.generateKeyPair()

        // Create a 20-byte token (what the device sends)
        let token = Data((0..<20).map { _ in UInt8.random(in: 0...255) })

        let signature = try auth.sign(token: token, with: keyPair.privateKeyData)

        // RSA 2048 signature should be 256 bytes
        XCTAssertEqual(signature.count, 256)
    }

    func testSaveAndLoadKeyPair() throws {
        let auth = ADBAuth()
        let tempDir = FileManager.default.temporaryDirectory
            .appendingPathComponent("questsync-test-\(UUID().uuidString)")

        defer { try? FileManager.default.removeItem(at: tempDir) }

        // Generate and save
        let original = try auth.generateKeyPair()
        try auth.saveKeyPair(original, to: tempDir)

        // Verify files exist
        XCTAssertTrue(FileManager.default.fileExists(
            atPath: tempDir.appendingPathComponent("adbkey").path))
        XCTAssertTrue(FileManager.default.fileExists(
            atPath: tempDir.appendingPathComponent("adbkey.pub").path))

        // Load and verify round-trip
        let loaded = try auth.loadKeyPair(from: tempDir)
        XCTAssertEqual(loaded.privateKeyData, original.privateKeyData)
        XCTAssertEqual(loaded.androidPublicKey, original.androidPublicKey)
    }

    func testFilePermissions() throws {
        let auth = ADBAuth()
        let tempDir = FileManager.default.temporaryDirectory
            .appendingPathComponent("questsync-test-\(UUID().uuidString)")

        defer { try? FileManager.default.removeItem(at: tempDir) }

        let keyPair = try auth.generateKeyPair()
        try auth.saveKeyPair(keyPair, to: tempDir)

        // Private key should have 0600 permissions
        let attrs = try FileManager.default.attributesOfItem(
            atPath: tempDir.appendingPathComponent("adbkey").path)
        let permissions = (attrs[.posixPermissions] as? Int) ?? 0
        XCTAssertEqual(permissions, 0o600)
    }

    func testLoadOrGenerateCreatesNewIfMissing() throws {
        let auth = ADBAuth()
        let tempDir = FileManager.default.temporaryDirectory
            .appendingPathComponent("questsync-test-\(UUID().uuidString)")

        defer { try? FileManager.default.removeItem(at: tempDir) }

        // Should generate new keys since directory doesn't exist yet
        let keyPair = try auth.loadOrGenerate(directory: tempDir)
        XCTAssertGreaterThan(keyPair.privateKeyData.count, 0)

        // Should load existing keys on second call
        let loaded = try auth.loadOrGenerate(directory: tempDir)
        XCTAssertEqual(loaded.privateKeyData, keyPair.privateKeyData)
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd /Users/ishanperera/Desktop/Projects/questprojects/questfiletransfer/QuestSyncKit && swift test`
Expected: Compilation errors — `ADBAuth`, `ADBKeyPair` not defined.

- [ ] **Step 3: Implement ADBAuth**

Create `QuestSyncKit/Sources/QuestSyncKit/ADB/ADBAuth.swift`:

```swift
import Foundation
import Security

/// Errors during ADB authentication operations.
public enum ADBAuthError: Error, Sendable {
    case keyGenerationFailed(OSStatus)
    case signingFailed(OSStatus)
    case keyNotFound
    case invalidKeyData
    case filePermissionError
}

/// An RSA keypair for ADB authentication.
public struct ADBKeyPair: Sendable {
    /// DER-encoded PKCS#1 RSAPrivateKey (as returned by SecKeyCopyExternalRepresentation).
    public let privateKeyData: Data
    /// Android-format public key (base64 struct + " user@host\0").
    public let androidPublicKey: Data
}

/// Handles RSA key generation, signing, and persistence for ADB authentication.
public struct ADBAuth: Sendable {

    public init() {}

    /// Generate a new RSA 2048-bit keypair.
    public func generateKeyPair() throws -> ADBKeyPair {
        let attributes: [String: Any] = [
            kSecAttrKeyType as String: kSecAttrKeyTypeRSA,
            kSecAttrKeySizeInBits as String: 2048
        ]

        var error: Unmanaged<CFError>?
        guard let privateKey = SecKeyCreateRandomKey(attributes as CFDictionary, &error) else {
            throw ADBAuthError.keyGenerationFailed(-1)
        }

        // Export private key as DER data
        var exportError: Unmanaged<CFError>?
        guard let privateKeyData = SecKeyCopyExternalRepresentation(privateKey, &exportError) as Data? else {
            throw ADBAuthError.keyGenerationFailed(-2)
        }

        // Get public key and format for Android
        guard let publicKey = SecKeyCopyPublicKey(privateKey) else {
            throw ADBAuthError.keyGenerationFailed(-3)
        }

        let androidPubKey = try formatAndroidPublicKey(publicKey)

        return ADBKeyPair(privateKeyData: privateKeyData, androidPublicKey: androidPubKey)
    }

    /// Sign a token with the private key (PKCS1 SHA1, as required by ADB protocol).
    /// ADB sends a 20-byte random token; we hash it with SHA-1 then sign with RSA PKCS#1 v1.5.
    public func sign(token: Data, with privateKeyData: Data) throws -> Data {
        let attributes: [String: Any] = [
            kSecAttrKeyType as String: kSecAttrKeyTypeRSA,
            kSecAttrKeyClass as String: kSecAttrKeyClassPrivate,
            kSecAttrKeySizeInBits as String: 2048
        ]

        var error: Unmanaged<CFError>?
        guard let privateKey = SecKeyCreateWithData(
            privateKeyData as CFData,
            attributes as CFDictionary,
            &error
        ) else {
            throw ADBAuthError.invalidKeyData
        }

        var signError: Unmanaged<CFError>?
        guard let signature = SecKeyCreateSignature(
            privateKey,
            .rsaSignatureMessagePKCS1v15SHA1,
            token as CFData,
            &signError
        ) as Data? else {
            throw ADBAuthError.signingFailed(-1)
        }

        return signature
    }

    /// Save a keypair to disk with secure permissions.
    public func saveKeyPair(_ keyPair: ADBKeyPair, to directory: URL) throws {
        let fm = FileManager.default

        if !fm.fileExists(atPath: directory.path) {
            try fm.createDirectory(at: directory, withIntermediateDirectories: true)
        }

        let privateKeyPath = directory.appendingPathComponent("adbkey")
        let publicKeyPath = directory.appendingPathComponent("adbkey.pub")

        // Write private key with 0600 permissions
        try keyPair.privateKeyData.write(to: privateKeyPath)
        try fm.setAttributes(
            [.posixPermissions: 0o600],
            ofItemAtPath: privateKeyPath.path
        )

        // Write public key
        try keyPair.androidPublicKey.write(to: publicKeyPath)
        try fm.setAttributes(
            [.posixPermissions: 0o600],
            ofItemAtPath: publicKeyPath.path
        )
    }

    /// Load a keypair from disk.
    public func loadKeyPair(from directory: URL) throws -> ADBKeyPair {
        let privateKeyPath = directory.appendingPathComponent("adbkey")
        let publicKeyPath = directory.appendingPathComponent("adbkey.pub")

        guard FileManager.default.fileExists(atPath: privateKeyPath.path) else {
            throw ADBAuthError.keyNotFound
        }

        let privateKeyData = try Data(contentsOf: privateKeyPath)
        let publicKeyData = try Data(contentsOf: publicKeyPath)

        return ADBKeyPair(privateKeyData: privateKeyData, androidPublicKey: publicKeyData)
    }

    /// Load existing keys or generate new ones if none exist.
    public func loadOrGenerate(directory: URL) throws -> ADBKeyPair {
        do {
            return try loadKeyPair(from: directory)
        } catch ADBAuthError.keyNotFound {
            let keyPair = try generateKeyPair()
            try saveKeyPair(keyPair, to: directory)
            return keyPair
        }
    }

    // MARK: - Android Public Key Format

    /// Convert a SecKey public key to Android's proprietary ADB public key format.
    ///
    /// Android's format is a base64-encoded struct:
    /// ```
    /// struct RSAPublicKey {
    ///     uint32_t len;        // Number of 32-bit words in modulus
    ///     uint32_t n0inv;      // -1 / n[0] mod 2^32
    ///     uint32_t n[64];      // Modulus (little-endian, 2048 bits)
    ///     uint32_t rr[64];     // R^2 mod n (Montgomery parameter)
    ///     uint32_t exponent;   // Public exponent (usually 65537)
    /// }
    /// ```
    /// Followed by ` user@hostname\0`
    private func formatAndroidPublicKey(_ publicKey: SecKey) throws -> Data {
        var error: Unmanaged<CFError>?
        guard let pubKeyData = SecKeyCopyExternalRepresentation(publicKey, &error) as Data? else {
            throw ADBAuthError.invalidKeyData
        }

        // Parse the DER-encoded RSA public key to extract modulus and exponent
        let (modulus, exponent) = try parseRSAPublicKeyDER(pubKeyData)

        // Build Android struct
        let wordCount = modulus.count / 4  // 64 words for 2048-bit key
        var androidKey = Data(capacity: 4 + 4 + 256 + 256 + 4)  // 524 bytes

        // len (number of 32-bit words)
        androidKey.appendLittleEndian(UInt32(wordCount))

        // n0inv: -1 / n[0] mod 2^32
        let n0 = modulusWord(modulus, at: 0)
        let n0inv = computeN0Inv(n0)
        androidKey.appendLittleEndian(n0inv)

        // n[]: modulus in little-endian word order
        for i in 0..<wordCount {
            androidKey.appendLittleEndian(modulusWord(modulus, at: i))
        }

        // rr[]: R^2 mod n (Montgomery parameter)
        let rr = computeRR(modulus: modulus, wordCount: wordCount)
        for word in rr {
            androidKey.appendLittleEndian(word)
        }

        // exponent
        let exp = exponent.withUnsafeBytes { bytes -> UInt32 in
            var value: UInt32 = 0
            for byte in bytes {
                value = (value << 8) | UInt32(byte)
            }
            return value
        }
        androidKey.appendLittleEndian(exp)

        // Base64 encode + append user@hostname identifier
        let base64 = androidKey.base64EncodedString()
        let hostname = ProcessInfo.processInfo.hostName
        let username = NSUserName()
        let identifier = " \(username)@\(hostname)\0"

        var result = Data(base64.utf8)
        result.append(Data(identifier.utf8))

        return result
    }

    /// Parse DER-encoded RSA public key to extract modulus and exponent bytes.
    private func parseRSAPublicKeyDER(_ der: Data) throws -> (modulus: Data, exponent: Data) {
        // SecKeyCopyExternalRepresentation for RSA returns the raw key in a simplified format:
        // SEQUENCE { INTEGER(modulus), INTEGER(exponent) }
        // We need to parse the ASN.1 DER encoding
        var index = der.startIndex

        // Outer SEQUENCE
        guard der[index] == 0x30 else { throw ADBAuthError.invalidKeyData }
        index = try skipDERLength(der, from: der.index(after: index))

        // Modulus INTEGER
        guard der[index] == 0x02 else { throw ADBAuthError.invalidKeyData }
        index = der.index(after: index)
        let (modulusLength, modulusStart) = try readDERLength(der, from: index)
        var modulus = der[modulusStart..<der.index(modulusStart, offsetBy: modulusLength)]
        // Strip leading zero byte if present (ASN.1 sign byte)
        if modulus.first == 0x00 {
            modulus = modulus.dropFirst()
        }
        index = der.index(modulusStart, offsetBy: modulusLength)

        // Exponent INTEGER
        guard der[index] == 0x02 else { throw ADBAuthError.invalidKeyData }
        index = der.index(after: index)
        let (exponentLength, exponentStart) = try readDERLength(der, from: index)
        let exponent = der[exponentStart..<der.index(exponentStart, offsetBy: exponentLength)]

        return (Data(modulus), Data(exponent))
    }

    private func readDERLength(_ data: Data, from index: Data.Index) throws -> (Int, Data.Index) {
        let byte = data[index]
        if byte < 0x80 {
            return (Int(byte), data.index(after: index))
        }
        let numBytes = Int(byte & 0x7F)
        var length = 0
        var idx = data.index(after: index)
        for _ in 0..<numBytes {
            length = (length << 8) | Int(data[idx])
            idx = data.index(after: idx)
        }
        return (length, idx)
    }

    private func skipDERLength(_ data: Data, from index: Data.Index) throws -> Data.Index {
        let (_, nextIndex) = try readDERLength(data, from: index)
        return nextIndex
    }

    /// Read a 32-bit word from the modulus at the given word index (little-endian order).
    private func modulusWord(_ modulus: Data, at wordIndex: Int) -> UInt32 {
        // Modulus is big-endian bytes. Word 0 is the least significant 4 bytes.
        let byteOffset = modulus.count - 4 - (wordIndex * 4)
        let start = modulus.startIndex + byteOffset
        return modulus[start..<start+4].withUnsafeBytes { ptr in
            let be = ptr.load(as: UInt32.self)
            // Bytes are big-endian in modulus, but we want the word value
            return UInt32(bigEndian: be)
        }
    }

    /// Compute n0inv = -1/n[0] mod 2^32 using the extended Euclidean algorithm.
    private func computeN0Inv(_ n0: UInt32) -> UInt32 {
        var result: UInt32 = 0
        var t: UInt32 = 0
        var i: UInt32 = 1
        for _ in 0..<32 {
            if (t & 1) == 0 {
                t = t &+ n0
                result = result &+ i
            }
            t >>= 1
            i <<= 1
        }
        return result
    }

    /// Compute R^2 mod n for Montgomery multiplication.
    private func computeRR(modulus: Data, wordCount: Int) -> [UInt32] {
        // R = 2^(wordCount * 32)
        // RR = R^2 mod n
        // We compute this by repeated doubling: start with R mod n, double it wordCount*32 times
        // For simplicity, use a basic big-number approach
        var rr = [UInt32](repeating: 0, count: wordCount)

        // Start with 1
        rr[0] = 1

        // Double wordCount * 64 times (to get R^2)
        for _ in 0..<(wordCount * 64) {
            // Left shift by 1
            var carry: UInt32 = 0
            for j in 0..<wordCount {
                let cur = UInt64(rr[j]) << 1 | UInt64(carry)
                rr[j] = UInt32(cur & 0xFFFFFFFF)
                carry = UInt32(cur >> 32)
            }

            // If rr >= modulus, subtract modulus
            if carry != 0 || compareWithModulus(rr, modulus: modulus, wordCount: wordCount) >= 0 {
                var borrow: Int64 = 0
                for j in 0..<wordCount {
                    let mWord = Int64(modulusWord(modulus, at: j))
                    let diff = Int64(rr[j]) - mWord + borrow
                    if diff < 0 {
                        rr[j] = UInt32(truncatingIfNeeded: diff &+ 0x100000000)
                        borrow = -1
                    } else {
                        rr[j] = UInt32(truncatingIfNeeded: diff)
                        borrow = 0
                    }
                }
            }
        }

        return rr
    }

    /// Compare rr with modulus. Returns -1, 0, or 1.
    private func compareWithModulus(_ rr: [UInt32], modulus: Data, wordCount: Int) -> Int {
        for i in stride(from: wordCount - 1, through: 0, by: -1) {
            let rrWord = rr[i]
            let mWord = modulusWord(modulus, at: i)
            if rrWord < mWord { return -1 }
            if rrWord > mWord { return 1 }
        }
        return 0
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd /Users/ishanperera/Desktop/Projects/questprojects/questfiletransfer/QuestSyncKit && swift test`
Expected: All tests pass (ADBMessageTests + MockTransportTests + ADBAuthTests).

- [ ] **Step 5: Commit**

```bash
git add QuestSyncKit/Sources/QuestSyncKit/ADB/ADBAuth.swift QuestSyncKit/Tests/QuestSyncKitTests/ADBAuthTests.swift
git commit -m "feat: implement ADB RSA authentication with Android public key format"
```

---

## Chunk 2: ADB Connection State Machine & USB Transport

### Task 5: ADB Connection State Machine

**Files:**
- Create: `QuestSyncKit/Sources/QuestSyncKit/ADB/ADBConnection.swift`
- Create: `QuestSyncKit/Tests/QuestSyncKitTests/ADBConnectionTests.swift`

The connection manages the protocol state machine:
1. Open transport
2. Send CNXN → receive AUTH (token)
3. Sign token → send AUTH (signature)
4. If not recognized: send AUTH (public key) → user approves on device
5. Receive CNXN → connected, parse device banner for maxdata and features

- [ ] **Step 1: Write failing tests for ADBConnection**

Create `QuestSyncKit/Tests/QuestSyncKitTests/ADBConnectionTests.swift`:

```swift
import XCTest
@testable import QuestSyncKit

final class ADBConnectionTests: XCTestCase {

    var transport: MockTransport!
    var auth: ADBAuth!
    var keyPair: ADBKeyPair!
    var tempDir: URL!

    override func setUp() async throws {
        transport = MockTransport()
        auth = ADBAuth()
        tempDir = FileManager.default.temporaryDirectory
            .appendingPathComponent("questsync-test-\(UUID().uuidString)")
        keyPair = try auth.loadOrGenerate(directory: tempDir)
    }

    override func tearDown() async throws {
        try? FileManager.default.removeItem(at: tempDir)
    }

    func testInitialStateIsDisconnected() {
        let connection = ADBConnection(transport: transport, keyDirectory: tempDir)
        XCTAssertEqual(connection.state, .disconnected)
    }

    func testSuccessfulConnectionWithKnownKey() async throws {
        let connection = ADBConnection(transport: transport, keyDirectory: tempDir)

        // Device will send: AUTH token, then CNXN (key already known)
        let token = Data((0..<20).map { _ in UInt8.random(in: 0...255) })
        transport.enqueueResponse(ADBMessage(command: .auth, arg0: 1, arg1: 0, data: token))
        transport.enqueueResponse(ADBMessage.connect(
            version: 0x01000001,
            maxData: 262144,
            systemIdentity: "device::ro.product.model=Quest 3;ro.product.name=eureka;\0"
        ))

        try await connection.connect()

        XCTAssertEqual(connection.state, .ready)
        XCTAssertEqual(connection.maxData, 262144)

        // Should have sent: CNXN, then AUTH signature
        XCTAssertEqual(transport.sentMessages.count, 2)
        XCTAssertEqual(transport.sentMessages[0].command, .connect)
        XCTAssertEqual(transport.sentMessages[1].command, .auth)
    }

    func testConnectionWithNewKeyRequiresApproval() async throws {
        let connection = ADBConnection(transport: transport, keyDirectory: tempDir)

        // Device sends: AUTH token, then AUTH token again (signature not recognized),
        // then CNXN (after user approves public key)
        let token = Data((0..<20).map { _ in UInt8.random(in: 0...255) })
        transport.enqueueResponse(ADBMessage(command: .auth, arg0: 1, arg1: 0, data: token))
        // Second AUTH means signature wasn't recognized — send public key
        transport.enqueueResponse(ADBMessage(command: .auth, arg0: 1, arg1: 0, data: token))
        // After public key sent, device responds with CNXN (user approved)
        transport.enqueueResponse(ADBMessage.connect(
            version: 0x01000001,
            maxData: 262144,
            systemIdentity: "device::\0"
        ))

        try await connection.connect()

        XCTAssertEqual(connection.state, .ready)
        // Should have sent: CNXN, AUTH signature, AUTH public key
        XCTAssertEqual(transport.sentMessages.count, 3)
        XCTAssertEqual(transport.sentMessages[0].command, .connect)
        XCTAssertEqual(transport.sentMessages[1].command, .auth) // signature
        XCTAssertEqual(transport.sentMessages[1].arg0, 2) // type = SIGNATURE
        XCTAssertEqual(transport.sentMessages[2].command, .auth) // public key
        XCTAssertEqual(transport.sentMessages[2].arg0, 3) // type = RSAPUBLICKEY
    }

    func testDisconnect() async throws {
        let connection = ADBConnection(transport: transport, keyDirectory: tempDir)

        let token = Data((0..<20).map { _ in UInt8.random(in: 0...255) })
        transport.enqueueResponse(ADBMessage(command: .auth, arg0: 1, arg1: 0, data: token))
        transport.enqueueResponse(ADBMessage.connect(
            version: 0x01000001, maxData: 262144, systemIdentity: "device::\0"
        ))

        try await connection.connect()
        XCTAssertEqual(connection.state, .ready)

        await connection.disconnect()
        XCTAssertEqual(connection.state, .disconnected)
        XCTAssertFalse(transport.isOpen)
    }

    func testParseDeviceBanner() async throws {
        let connection = ADBConnection(transport: transport, keyDirectory: tempDir)

        let token = Data((0..<20).map { _ in UInt8.random(in: 0...255) })
        transport.enqueueResponse(ADBMessage(command: .auth, arg0: 1, arg1: 0, data: token))
        transport.enqueueResponse(ADBMessage.connect(
            version: 0x01000001,
            maxData: 262144,
            systemIdentity: "device::ro.product.model=Quest 3;ro.product.name=eureka;\0"
        ))

        try await connection.connect()

        XCTAssertEqual(connection.deviceModel, "Quest 3")
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd /Users/ishanperera/Desktop/Projects/questprojects/questfiletransfer/QuestSyncKit && swift test`
Expected: Compilation errors — `ADBConnection` not defined.

- [ ] **Step 3: Implement ADBConnection**

Create `QuestSyncKit/Sources/QuestSyncKit/ADB/ADBConnection.swift`:

```swift
import Foundation
import os

/// ADB connection states.
public enum ADBConnectionState: Sendable, Equatable {
    case disconnected
    case connecting
    case authenticating
    case waitingForApproval
    case ready
}

/// Errors during ADB connection.
public enum ADBConnectionError: Error, Sendable {
    case unexpectedMessage(ADBCommand)
    case authenticationFailed
    case connectionRejected
    case alreadyConnected
    case notConnected
}

/// Manages an ADB connection to a device over a transport.
///
/// Handles the CNXN → AUTH → CNXN handshake, including RSA key
/// generation and the "Allow USB debugging?" approval flow.
public final class ADBConnection: @unchecked Sendable {
    private let transport: ADBTransport
    private let auth: ADBAuth
    private let keyDirectory: URL
    private let logger = Logger(subsystem: "com.questsync.app", category: "adb")

    private let lock = NSLock()
    private var _state: ADBConnectionState = .disconnected
    private var _maxData: UInt32 = 4096
    private var _deviceBanner: [String: String] = [:]

    /// Current connection state.
    public var state: ADBConnectionState {
        lock.withLock { _state }
    }

    /// Maximum data payload size negotiated with the device.
    public var maxData: UInt32 {
        lock.withLock { _maxData }
    }

    /// Device model name parsed from the connection banner.
    public var deviceModel: String? {
        lock.withLock { _deviceBanner["ro.product.model"] }
    }

    /// ADB protocol version.
    private static let protocolVersion: UInt32 = 0x01000001
    /// Default max payload size we advertise.
    private static let defaultMaxData: UInt32 = 262144 // 256KB

    public init(transport: ADBTransport, keyDirectory: URL) {
        self.transport = transport
        self.auth = ADBAuth()
        self.keyDirectory = keyDirectory
    }

    /// Connect to the device. Performs the full CNXN → AUTH → CNXN handshake.
    public func connect() async throws {
        let currentState = lock.withLock { _state }
        guard currentState == .disconnected else {
            throw ADBConnectionError.alreadyConnected
        }

        lock.withLock { _state = .connecting }
        logger.info("Opening transport...")

        try await transport.open()

        // Load or generate RSA keys
        let keyPair = try auth.loadOrGenerate(directory: keyDirectory)

        // Send CNXN
        let systemIdentity = "host::\0"
        let cnxn = ADBMessage.connect(
            version: Self.protocolVersion,
            maxData: Self.defaultMaxData,
            systemIdentity: systemIdentity
        )
        try await transport.send(cnxn)
        logger.info("Sent CNXN, waiting for response...")

        lock.withLock { _state = .authenticating }

        // Expect AUTH token from device
        let response = try await transport.receive()

        switch response.command {
        case .auth:
            // Device sent AUTH with type=1 (TOKEN)
            try await handleAuth(response, keyPair: keyPair)

        case .connect:
            // Device accepted immediately (rare, but possible with pre-authorized keys)
            try parseDeviceBanner(response)
            lock.withLock { _state = .ready }

        default:
            throw ADBConnectionError.unexpectedMessage(response.command)
        }
    }

    /// Disconnect from the device.
    public func disconnect() async {
        await transport.close()
        lock.withLock {
            _state = .disconnected
            _deviceBanner = [:]
            _maxData = 4096
        }
        logger.info("Disconnected")
    }

    // MARK: - Authentication

    private func handleAuth(_ authMessage: ADBMessage, keyPair: ADBKeyPair) async throws {
        // Sign the token
        let signature = try auth.sign(token: authMessage.data, with: keyPair.privateKeyData)
        let authSig = ADBMessage.authSignature(signature)
        try await transport.send(authSig)
        logger.info("Sent AUTH signature")

        // Wait for response
        let response = try await transport.receive()

        switch response.command {
        case .connect:
            // Signature accepted — we're authorized
            try parseDeviceBanner(response)
            lock.withLock { _state = .ready }
            logger.info("Connected (known key)")

        case .auth:
            // Signature not recognized — send public key for user approval
            lock.withLock { _state = .waitingForApproval }
            logger.info("Key not recognized, sending public key for approval...")

            let authPubKey = ADBMessage.authPublicKey(keyPair.androidPublicKey)
            try await transport.send(authPubKey)

            // Wait for user to approve on device → CNXN response
            let approval = try await transport.receive()
            guard approval.command == .connect else {
                throw ADBConnectionError.authenticationFailed
            }

            try parseDeviceBanner(approval)
            lock.withLock { _state = .ready }
            logger.info("Connected (new key approved)")

        default:
            throw ADBConnectionError.unexpectedMessage(response.command)
        }
    }

    // MARK: - Banner Parsing

    private func parseDeviceBanner(_ cnxn: ADBMessage) throws {
        lock.withLock {
            _maxData = cnxn.arg1
        }

        // Banner format: "device::key1=value1;key2=value2;\0"
        guard let banner = String(data: cnxn.data, encoding: .utf8) else { return }

        // Split on "::" to get properties
        let parts = banner.split(separator: "::", maxSplits: 1)
        guard parts.count == 2 else { return }

        let properties = parts[1]
            .trimmingCharacters(in: .init(charactersIn: "\0"))
            .split(separator: ";")

        var parsed: [String: String] = [:]
        for prop in properties {
            let kv = prop.split(separator: "=", maxSplits: 1)
            if kv.count == 2 {
                parsed[String(kv[0])] = String(kv[1])
            }
        }

        lock.withLock {
            _deviceBanner = parsed
        }
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd /Users/ishanperera/Desktop/Projects/questprojects/questfiletransfer/QuestSyncKit && swift test`
Expected: All tests pass.

- [ ] **Step 5: Commit**

```bash
git add QuestSyncKit/Sources/QuestSyncKit/ADB/ADBConnection.swift QuestSyncKit/Tests/QuestSyncKitTests/ADBConnectionTests.swift
git commit -m "feat: implement ADB connection state machine with auth handshake"
```

---

### Task 6: Device Model

**Files:**
- Create: `QuestSyncKit/Sources/QuestSyncKit/Device/QuestDevice.swift`

- [ ] **Step 1: Implement QuestDevice model**

Create `QuestSyncKit/Sources/QuestSyncKit/Device/QuestDevice.swift`:

```swift
import Foundation

/// Connection state of a Quest device.
public enum QuestDeviceState: Sendable, Equatable {
    case detected           // USB device found, ADB not yet attempted
    case connecting         // ADB handshake in progress
    case waitingForApproval // User needs to approve on headset
    case connected          // ADB authenticated and ready
    case noADBInterface     // Device found but no ADB interface (Developer Mode off)
    case error(String)      // Connection error
}

/// Represents a connected Meta Quest device.
public struct QuestDevice: Identifiable, Sendable {
    public let id: String          // USB serial number
    public let vendorID: UInt16    // Should be 0x2833 (Meta)
    public let productID: UInt16
    public var state: QuestDeviceState
    public var model: String?      // e.g., "Quest 3", populated after ADB connection
    public var serial: String?     // ADB serial, populated after connection

    /// Meta's USB vendor ID.
    public static let metaVendorID: UInt16 = 0x2833

    /// ADB USB interface identifiers.
    public static let adbInterfaceClass: UInt8 = 0xFF
    public static let adbInterfaceSubClass: UInt8 = 0x42
    public static let adbInterfaceProtocol: UInt8 = 0x01

    public init(
        id: String,
        vendorID: UInt16,
        productID: UInt16,
        state: QuestDeviceState = .detected
    ) {
        self.id = id
        self.vendorID = vendorID
        self.productID = productID
        self.state = state
    }

    /// Whether this device appears to be a Meta Quest.
    public var isMetaDevice: Bool {
        vendorID == Self.metaVendorID
    }
}
```

- [ ] **Step 2: Verify it compiles**

Run: `cd /Users/ishanperera/Desktop/Projects/questprojects/questfiletransfer/QuestSyncKit && swift build`
Expected: Build succeeds.

- [ ] **Step 3: Commit**

```bash
git add QuestSyncKit/Sources/QuestSyncKit/Device/QuestDevice.swift
git commit -m "feat: add QuestDevice model with connection states"
```

---

### Task 7: USB Transport (IOKit)

**Files:**
- Create: `QuestSyncKit/Sources/QuestSyncKit/Transport/USBTransport.swift`

This is the most complex task in Phase 1. IOKit is a C API used from Swift. The USBTransport must:
1. Take an IOKit USB device reference
2. Find and claim the ADB interface (class 0xFF, subclass 0x42, protocol 0x01)
3. Detach kernel driver if needed
4. Find bulk IN and bulk OUT endpoints
5. Read/write ADB messages over the bulk endpoints

Note: This cannot be unit tested without hardware. It will be integration-tested with a real Quest in Task 8.

- [ ] **Step 1: Implement USBTransport**

Create `QuestSyncKit/Sources/QuestSyncKit/Transport/USBTransport.swift`:

```swift
import Foundation
import IOKit
import IOKit.usb
import os

/// IOKit-based USB transport for ADB communication with Quest devices.
///
/// Handles claiming the ADB USB interface, detaching kernel drivers,
/// and performing bulk reads/writes of ADB messages.
public final class USBTransport: ADBTransport, @unchecked Sendable {
    private let logger = Logger(subsystem: "com.questsync.app", category: "device")

    private let lock = NSLock()
    private var deviceInterface: UnsafeMutablePointer<UnsafeMutablePointer<IOUSBDeviceInterface>>?
    private var interfaceInterface: UnsafeMutablePointer<UnsafeMutablePointer<IOUSBInterfaceInterface>>?
    private var bulkInPipe: UInt8 = 0
    private var bulkOutPipe: UInt8 = 0
    private var _isOpen = false

    private let usbDevice: io_object_t

    public var isOpen: Bool {
        lock.withLock { _isOpen }
    }

    /// Initialize with an IOKit USB device reference.
    /// The caller is responsible for retaining the io_object_t until this transport is closed.
    public init(device: io_object_t) {
        self.usbDevice = device
    }

    public func open() async throws {
        try lock.withLock {
            guard !_isOpen else { return }

            // Get device interface
            let deviceIntf = try Self.getDeviceInterface(usbDevice)
            self.deviceInterface = deviceIntf

            // Open the device
            let openResult = deviceIntf.pointee.pointee.USBDeviceOpen(deviceIntf)
            guard openResult == kIOReturnSuccess else {
                throw ADBTransportError.connectionLost
            }

            // Find and claim ADB interface
            let (intfInterface, inPipe, outPipe) = try Self.findADBInterface(deviceIntf)
            self.interfaceInterface = intfInterface
            self.bulkInPipe = inPipe
            self.bulkOutPipe = outPipe

            _isOpen = true
            logger.info("USB transport opened: bulkIn=\(inPipe), bulkOut=\(outPipe)")
        }
    }

    public func close() async {
        lock.withLock {
            if let intfInterface = interfaceInterface {
                intfInterface.pointee.pointee.USBInterfaceClose(intfInterface)
                intfInterface.pointee.pointee.Release(intfInterface)
                self.interfaceInterface = nil
            }
            if let deviceIntf = deviceInterface {
                deviceIntf.pointee.pointee.USBDeviceClose(deviceIntf)
                deviceIntf.pointee.pointee.Release(deviceIntf)
                self.deviceInterface = nil
            }
            _isOpen = false
        }
        logger.info("USB transport closed")
    }

    public func send(_ message: ADBMessage) async throws {
        let data = message.serialize()
        try await bulkWrite(data)
    }

    public func receive() async throws -> ADBMessage {
        // Read 24-byte header
        let headerData = try await bulkReadExact(count: ADBMessage.headerSize)
        let header = try ADBMessage.deserializeHeader(from: headerData)

        let dataLength = header.expectedDataLength ?? 0
        if dataLength > 0 {
            let payload = try await bulkReadExact(count: Int(dataLength))
            return ADBMessage(command: header.command, arg0: header.arg0, arg1: header.arg1, data: payload)
        } else {
            return ADBMessage(command: header.command, arg0: header.arg0, arg1: header.arg1, data: Data())
        }
    }

    // MARK: - Bulk I/O

    private func bulkWrite(_ data: Data) async throws {
        try lock.withLock {
            guard _isOpen, let intfInterface = interfaceInterface else {
                throw ADBTransportError.notConnected
            }

            let result = data.withUnsafeBytes { ptr -> IOReturn in
                let mutablePtr = UnsafeMutablePointer(
                    mutating: ptr.baseAddress!.assumingMemoryBound(to: UInt8.self)
                )
                return intfInterface.pointee.pointee.WritePipe(
                    intfInterface,
                    bulkOutPipe,
                    mutablePtr,
                    UInt32(data.count)
                )
            }

            guard result == kIOReturnSuccess else {
                logger.error("Bulk write failed: \(result)")
                throw ADBTransportError.connectionLost
            }
        }
    }

    /// Read exactly `count` bytes, looping to handle partial USB reads.
    private func bulkReadExact(count: Int) async throws -> Data {
        var accumulated = Data()
        accumulated.reserveCapacity(count)

        while accumulated.count < count {
            let remaining = count - accumulated.count
            let chunk = try lock.withLock { () -> Data in
                guard _isOpen, let intfInterface = interfaceInterface else {
                    throw ADBTransportError.notConnected
                }

                var buffer = Data(count: remaining)
                var bytesRead = UInt32(remaining)

                let result = buffer.withUnsafeMutableBytes { ptr -> IOReturn in
                    return intfInterface.pointee.pointee.ReadPipe(
                        intfInterface,
                        bulkInPipe,
                        ptr.baseAddress!,
                        &bytesRead
                    )
                }

                guard result == kIOReturnSuccess else {
                    logger.error("Bulk read failed: \(result)")
                    throw ADBTransportError.connectionLost
                }

                return Data(buffer.prefix(Int(bytesRead)))
            }

            if chunk.isEmpty {
                throw ADBTransportError.connectionLost
            }
            accumulated.append(chunk)
        }

        return accumulated
    }

    // MARK: - IOKit Helpers

    private static func getDeviceInterface(
        _ device: io_object_t
    ) throws -> UnsafeMutablePointer<UnsafeMutablePointer<IOUSBDeviceInterface>> {
        var plugInInterface: UnsafeMutablePointer<UnsafeMutablePointer<IOCFPlugInInterface>?>?
        var score: Int32 = 0

        let kr = IOCreatePlugInInterfaceForService(
            device,
            kIOUSBDeviceUserClientTypeID,
            kIOCFPlugInInterfaceID,
            &plugInInterface,
            &score
        )
        guard kr == kIOReturnSuccess, let plugIn = plugInInterface?.pointee else {
            throw ADBTransportError.connectionLost
        }
        defer { plugIn.pointee.Release(plugIn) }

        var deviceInterfacePtr: UnsafeMutableRawPointer?
        let result = withUnsafeMutablePointer(to: &deviceInterfacePtr) { ptr in
            plugIn.pointee.QueryInterface(
                plugIn,
                CFUUIDGetUUIDBytes(kIOUSBDeviceInterfaceID),
                ptr
            )
        }
        guard result == S_OK, let rawPtr = deviceInterfacePtr else {
            throw ADBTransportError.connectionLost
        }

        return rawPtr.assumingMemoryBound(
            to: UnsafeMutablePointer<IOUSBDeviceInterface>.self
        )
    }

    private static func findADBInterface(
        _ deviceInterface: UnsafeMutablePointer<UnsafeMutablePointer<IOUSBDeviceInterface>>
    ) throws -> (
        UnsafeMutablePointer<UnsafeMutablePointer<IOUSBInterfaceInterface>>,
        UInt8,  // bulkInPipe
        UInt8   // bulkOutPipe
    ) {
        // Create interface request matching ADB class/subclass/protocol
        var request = IOUSBFindInterfaceRequest(
            bInterfaceClass: UInt16(QuestDevice.adbInterfaceClass),
            bInterfaceSubClass: UInt16(QuestDevice.adbInterfaceSubClass),
            bInterfaceProtocol: UInt16(QuestDevice.adbInterfaceProtocol),
            bAlternateSetting: UInt16(kIOUSBFindInterfaceDontCare)
        )

        var iterator: io_iterator_t = 0
        let kr = deviceInterface.pointee.pointee.CreateInterfaceIterator(
            deviceInterface, &request, &iterator
        )
        guard kr == kIOReturnSuccess else {
            throw ADBTransportError.connectionLost
        }
        defer { IOObjectRelease(iterator) }

        let interfaceService = IOIteratorNext(iterator)
        guard interfaceService != 0 else {
            throw ADBTransportError.connectionLost
        }
        defer { IOObjectRelease(interfaceService) }

        // Get interface interface
        var plugInInterface: UnsafeMutablePointer<UnsafeMutablePointer<IOCFPlugInInterface>?>?
        var score: Int32 = 0
        let plugKr = IOCreatePlugInInterfaceForService(
            interfaceService,
            kIOUSBInterfaceUserClientTypeID,
            kIOCFPlugInInterfaceID,
            &plugInInterface,
            &score
        )
        guard plugKr == kIOReturnSuccess, let plugIn = plugInInterface?.pointee else {
            throw ADBTransportError.connectionLost
        }
        defer { plugIn.pointee.Release(plugIn) }

        var intfPtr: UnsafeMutableRawPointer?
        let qResult = withUnsafeMutablePointer(to: &intfPtr) { ptr in
            plugIn.pointee.QueryInterface(
                plugIn,
                CFUUIDGetUUIDBytes(kIOUSBInterfaceInterfaceID),
                ptr
            )
        }
        guard qResult == S_OK, let rawIntf = intfPtr else {
            throw ADBTransportError.connectionLost
        }

        let intfInterface = rawIntf.assumingMemoryBound(
            to: UnsafeMutablePointer<IOUSBInterfaceInterface>.self
        )

        // Open the interface
        let openResult = intfInterface.pointee.pointee.USBInterfaceOpen(intfInterface)
        guard openResult == kIOReturnSuccess else {
            throw ADBTransportError.connectionLost
        }

        // Find bulk endpoints
        var numEndpoints: UInt8 = 0
        intfInterface.pointee.pointee.GetNumEndpoints(intfInterface, &numEndpoints)

        var bulkIn: UInt8 = 0
        var bulkOut: UInt8 = 0

        for i in 1...numEndpoints {
            var direction: UInt8 = 0
            var number: UInt8 = 0
            var transferType: UInt8 = 0
            var maxPacketSize: UInt16 = 0
            var interval: UInt8 = 0

            intfInterface.pointee.pointee.GetPipeProperties(
                intfInterface, i,
                &direction, &number, &transferType,
                &maxPacketSize, &interval
            )

            // Bulk transfer type = 2
            if transferType == 2 {
                if direction == 1 { // IN
                    bulkIn = i
                } else { // OUT
                    bulkOut = i
                }
            }
        }

        guard bulkIn != 0 && bulkOut != 0 else {
            intfInterface.pointee.pointee.USBInterfaceClose(intfInterface)
            throw ADBTransportError.connectionLost
        }

        return (intfInterface, bulkIn, bulkOut)
    }
}
```

- [ ] **Step 2: Verify it compiles**

Run: `cd /Users/ishanperera/Desktop/Projects/questprojects/questfiletransfer/QuestSyncKit && swift build`
Expected: Build succeeds. (Note: IOKit APIs may need conditional compilation or may not all be available via SPM — if there are import issues, we'll address them.)

- [ ] **Step 3: Commit**

```bash
git add QuestSyncKit/Sources/QuestSyncKit/Transport/USBTransport.swift
git commit -m "feat: implement IOKit USB transport for ADB communication"
```

---

### Task 8: Device Manager (IOKit Notifications)

**Files:**
- Create: `QuestSyncKit/Sources/QuestSyncKit/Device/DeviceManager.swift`

The DeviceManager registers for IOKit USB notifications and emits device connect/disconnect events. It matches on Meta's vendor ID, then checks for the ADB interface to determine if Developer Mode is enabled.

- [ ] **Step 1: Implement DeviceManager**

Create `QuestSyncKit/Sources/QuestSyncKit/Device/DeviceManager.swift`:

```swift
import Foundation
import IOKit
import IOKit.usb
import Combine
import os

/// Monitors USB bus for Meta Quest devices and publishes connect/disconnect events.
public final class DeviceManager: ObservableObject, @unchecked Sendable {
    private let logger = Logger(subsystem: "com.questsync.app", category: "device")

    /// Currently detected devices.
    @Published public private(set) var devices: [QuestDevice] = []

    private var notifyPort: IONotificationPortRef?
    private var addedIterator: io_iterator_t = 0
    private var removedIterator: io_iterator_t = 0
    private let lock = NSLock()

    public init() {}

    /// Start monitoring for USB device connections.
    /// Must be called from the main thread (IOKit notifications use RunLoop).
    public func startMonitoring() {
        let matchingDict = IOServiceMatching(kIOUSBDeviceClassName) as NSMutableDictionary
        matchingDict[kUSBVendorID] = QuestDevice.metaVendorID

        notifyPort = IONotificationPortCreate(kIOMainPortDefault)
        guard let notifyPort = notifyPort else {
            logger.error("Failed to create IONotificationPort")
            return
        }

        let runLoopSource = IONotificationPortGetRunLoopSource(notifyPort).takeUnretainedValue()
        CFRunLoopAddSource(CFRunLoopGetMain(), runLoopSource, .defaultMode)

        // Register for device additions
        let selfPtr = Unmanaged.passUnretained(self).toOpaque()

        let addCallback: IOServiceMatchingCallback = { refcon, iterator in
            let manager = Unmanaged<DeviceManager>.fromOpaque(refcon!).takeUnretainedValue()
            manager.handleDeviceAdded(iterator: iterator)
        }

        // Need two copies of matching dict — IOKit consumes them
        let addDict = matchingDict.mutableCopy() as! NSMutableDictionary
        let removeDict = matchingDict.mutableCopy() as! NSMutableDictionary

        var kr = IOServiceAddMatchingNotification(
            notifyPort,
            kIOFirstMatchNotification,
            addDict,
            addCallback,
            selfPtr,
            &addedIterator
        )

        if kr == kIOReturnSuccess {
            // Drain existing devices
            handleDeviceAdded(iterator: addedIterator)
        } else {
            logger.error("Failed to register for device additions: \(kr)")
        }

        // Register for device removals
        let removeCallback: IOServiceMatchingCallback = { refcon, iterator in
            let manager = Unmanaged<DeviceManager>.fromOpaque(refcon!).takeUnretainedValue()
            manager.handleDeviceRemoved(iterator: iterator)
        }

        kr = IOServiceAddMatchingNotification(
            notifyPort,
            kIOTerminatedNotification,
            removeDict,
            removeCallback,
            selfPtr,
            &removedIterator
        )

        if kr == kIOReturnSuccess {
            // Drain existing
            handleDeviceRemoved(iterator: removedIterator)
        } else {
            logger.error("Failed to register for device removals: \(kr)")
        }

        logger.info("USB device monitoring started")
    }

    /// Stop monitoring for USB devices.
    public func stopMonitoring() {
        if addedIterator != 0 {
            IOObjectRelease(addedIterator)
            addedIterator = 0
        }
        if removedIterator != 0 {
            IOObjectRelease(removedIterator)
            removedIterator = 0
        }
        if let notifyPort = notifyPort {
            IONotificationPortDestroy(notifyPort)
            self.notifyPort = nil
        }
        logger.info("USB device monitoring stopped")
    }

    deinit {
        stopMonitoring()
        // Release all retained device refs
        for (_, ref) in deviceRefs {
            IOObjectRelease(ref)
        }
        deviceRefs.removeAll()
    }

    // MARK: - IOKit Callbacks

    /// IOKit device references for detected devices, keyed by device ID.
    /// Used to create USBTransport instances for ADB connections.
    private var deviceRefs: [String: io_object_t] = [:]

    /// Get the IOKit device reference for a detected device.
    /// Caller must NOT release the returned io_object_t — it is owned by DeviceManager.
    public func ioDevice(for deviceID: String) -> io_object_t? {
        lock.withLock { deviceRefs[deviceID] }
    }

    private func handleDeviceAdded(iterator: io_iterator_t) {
        while case let device = IOIteratorNext(iterator), device != 0 {
            let vendorID = Self.getDeviceProperty(device, key: kUSBVendorID) ?? 0
            let productID = Self.getDeviceProperty(device, key: kUSBProductID) ?? 0
            let serial = Self.getStringProperty(device, key: kUSBSerialNumberString) ?? UUID().uuidString

            let hasADB = Self.hasADBInterface(device)
            let state: QuestDeviceState = hasADB ? .detected : .noADBInterface

            let questDevice = QuestDevice(
                id: serial,
                vendorID: UInt16(vendorID),
                productID: UInt16(productID),
                state: state
            )

            lock.withLock {
                if !devices.contains(where: { $0.id == questDevice.id }) {
                    devices.append(questDevice)
                    // Retain the io_object_t so we can use it for USBTransport later
                    IOObjectRetain(device)
                    deviceRefs[serial] = device
                } else {
                    IOObjectRelease(device)
                }
            }

            logger.info("Quest device detected: vendorID=\(vendorID), productID=\(productID), adb=\(hasADB)")
        }
    }

    private func handleDeviceRemoved(iterator: io_iterator_t) {
        while case let device = IOIteratorNext(iterator), device != 0 {
            // Find and remove the matching device from our list
            lock.withLock {
                if let index = devices.firstIndex(where: { deviceRefs[$0.id] == device }) {
                    let removed = devices.remove(at: index)
                    if let ref = deviceRefs.removeValue(forKey: removed.id) {
                        IOObjectRelease(ref)
                    }
                }
            }
            IOObjectRelease(device)
        }
        logger.info("USB device removed")
    }

    // MARK: - IOKit Property Helpers

    private static func getDeviceProperty(_ device: io_object_t, key: String) -> Int? {
        guard let value = IORegistryEntryCreateCFProperty(
            device, key as CFString, kCFAllocatorDefault, 0
        )?.takeRetainedValue() else { return nil }
        return (value as? NSNumber)?.intValue
    }

    private static func getStringProperty(_ device: io_object_t, key: String) -> String? {
        guard let value = IORegistryEntryCreateCFProperty(
            device, key as CFString, kCFAllocatorDefault, 0
        )?.takeRetainedValue() else { return nil }
        return value as? String
    }

    /// Check if a USB device has an ADB interface (class 0xFF, subclass 0x42, protocol 0x01).
    private static func hasADBInterface(_ device: io_object_t) -> Bool {
        var iterator: io_iterator_t = 0
        let kr = IORegistryEntryGetChildIterator(device, kIOServicePlane, &iterator)
        guard kr == kIOReturnSuccess else { return false }
        defer { IOObjectRelease(iterator) }

        while case let child = IOIteratorNext(iterator), child != 0 {
            defer { IOObjectRelease(child) }

            let intfClass = getDeviceProperty(child, key: kUSBInterfaceClass)
            let intfSubClass = getDeviceProperty(child, key: kUSBInterfaceSubClass)
            let intfProtocol = getDeviceProperty(child, key: kUSBInterfaceProtocol)

            if intfClass == Int(QuestDevice.adbInterfaceClass) &&
               intfSubClass == Int(QuestDevice.adbInterfaceSubClass) &&
               intfProtocol == Int(QuestDevice.adbInterfaceProtocol) {
                return true
            }
        }

        return false
    }
}
```

- [ ] **Step 2: Verify it compiles**

Run: `cd /Users/ishanperera/Desktop/Projects/questprojects/questfiletransfer/QuestSyncKit && swift build`
Expected: Build succeeds.

- [ ] **Step 3: Commit**

```bash
git add QuestSyncKit/Sources/QuestSyncKit/Device/DeviceManager.swift
git commit -m "feat: implement DeviceManager with IOKit USB notifications"
```

---

## Chunk 3: Debug Harness & Integration Test

### Task 9: Minimal Debug App

**Files:**
- Create: `QuestSync/QuestSyncApp.swift`
- Create: `QuestSync/ContentView.swift`

A minimal SwiftUI app that shows detected devices and attempts an ADB connection. This serves as the Phase 1 "Hello Quest" integration test.

Note: This requires an Xcode project. For Phase 1, we'll create the app files and build/run through Xcode manually (SPM doesn't support IOKit entitlements for apps).

- [ ] **Step 1: Create the app entry point**

Create directory and file — `QuestSync/QuestSyncApp.swift`:

```swift
import SwiftUI

@main
struct QuestSyncApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}
```

- [ ] **Step 2: Create the debug ContentView**

Create `QuestSync/ContentView.swift`:

```swift
import SwiftUI
import QuestSyncKit

struct ContentView: View {
    @StateObject private var deviceManager = DeviceManager()
    @State private var logMessages: [String] = []
    @State private var isConnecting = false

    var body: some View {
        VStack(alignment: .leading, spacing: 16) {
            // Header
            Text("QuestSync — Debug Harness")
                .font(.title)
                .padding(.bottom, 4)

            // Device status
            GroupBox("Devices") {
                if deviceManager.devices.isEmpty {
                    Text("No Quest devices detected. Plug in your Quest 3 via USB.")
                        .foregroundStyle(.secondary)
                        .padding()
                } else {
                    ForEach(deviceManager.devices) { device in
                        HStack {
                            Circle()
                                .fill(statusColor(for: device.state))
                                .frame(width: 10, height: 10)
                            Text("Quest (\(String(format: "0x%04X", device.productID)))")
                            Spacer()
                            Text(stateDescription(device.state))
                                .foregroundStyle(.secondary)

                            if device.state == .detected {
                                Button("Connect") {
                                    Task { await attemptConnection(device: device) }
                                }
                                .disabled(isConnecting)
                            }
                        }
                        .padding(.vertical, 4)
                    }
                }
            }

            // Connection log
            GroupBox("Log") {
                ScrollView {
                    VStack(alignment: .leading, spacing: 2) {
                        ForEach(logMessages.indices, id: \.self) { i in
                            Text(logMessages[i])
                                .font(.system(.caption, design: .monospaced))
                                .textSelection(.enabled)
                        }
                    }
                    .frame(maxWidth: .infinity, alignment: .leading)
                }
                .frame(height: 200)
            }

            Spacer()
        }
        .padding()
        .frame(minWidth: 500, minHeight: 400)
        .onAppear {
            deviceManager.startMonitoring()
            log("Device monitoring started. Waiting for Quest...")
        }
    }

    private func attemptConnection(device: QuestDevice) async {
        isConnecting = true
        log("Attempting ADB connection to \(device.id)...")

        guard let ioDevice = deviceManager.ioDevice(for: device.id) else {
            log("ERROR: No IOKit reference for device \(device.id)")
            isConnecting = false
            return
        }

        let transport = USBTransport(device: ioDevice)
        let keyDir = FileManager.default.homeDirectoryForCurrentUser
            .appendingPathComponent(".questsync")
        let connection = ADBConnection(transport: transport, keyDirectory: keyDir)

        do {
            try await connection.connect()
            log("Connected! State: \(connection.state)")
            if let model = connection.deviceModel {
                log("Device model: \(model)")
            }
            log("Max data payload: \(connection.maxData) bytes")
            log("Phase 1 milestone COMPLETE: Hello Quest!")

            // Clean up
            await connection.disconnect()
            log("Disconnected.")
        } catch {
            log("ERROR: \(error)")
        }

        isConnecting = false
    }

    private func log(_ message: String) {
        let timestamp = DateFormatter.localizedString(
            from: Date(), dateStyle: .none, timeStyle: .medium
        )
        logMessages.append("[\(timestamp)] \(message)")
    }

    private func statusColor(for state: QuestDeviceState) -> Color {
        switch state {
        case .connected: return .green
        case .connecting, .waitingForApproval: return .yellow
        case .detected: return .blue
        case .noADBInterface: return .orange
        case .error: return .red
        }
    }

    private func stateDescription(_ state: QuestDeviceState) -> String {
        switch state {
        case .detected: return "Detected (ADB available)"
        case .connecting: return "Connecting..."
        case .waitingForApproval: return "Approve on headset"
        case .connected: return "Connected"
        case .noADBInterface: return "Developer Mode off"
        case .error(let msg): return "Error: \(msg)"
        }
    }
}
```

- [ ] **Step 3: Verify the package still builds**

Run: `cd /Users/ishanperera/Desktop/Projects/questprojects/questfiletransfer/QuestSyncKit && swift build`
Expected: Build succeeds (the app files won't be built by SPM — they'll use Xcode).

- [ ] **Step 4: Commit**

```bash
mkdir -p QuestSync
git add QuestSync/
git commit -m "feat: add minimal SwiftUI debug harness for Phase 1 testing"
```

---

### Task 10: Integration Test with Real Hardware

**Files:**
- No new files — this is a manual testing task

This task verifies the Phase 1 milestone with a real Quest 3.

- [ ] **Step 1: Create Xcode project**

Open Xcode → File → New → Project → macOS App
- Product Name: QuestSync
- Organization: com.questsync
- Interface: SwiftUI
- Add QuestSyncKit as a local Swift Package dependency
- Set deployment target to macOS 13

- [ ] **Step 2: Configure entitlements**

In the Xcode project, add an entitlements file with:
- `com.apple.security.device.usb` (USB device access)
- Disable App Sandbox for Phase 1 development

- [ ] **Step 3: Build and run the debug harness**

Run: Build and run in Xcode (Cmd+R)
Expected: App window opens showing "No Quest devices detected"

- [ ] **Step 4: Plug in Quest 3**

Connect Quest 3 via USB-C cable.
Expected: Device appears in the device list with status "Detected (ADB available)" or "Developer Mode off"

- [ ] **Step 5: Verify Phase 1 milestone**

If device shows "Detected (ADB available)":
- Click "Connect"
- Put on Quest headset
- Approve "Allow USB debugging?" prompt
- App should show connection success in log

If device shows "Developer Mode off":
- This confirms the IOKit detection and interface enumeration is working correctly
- Developer Mode needs to be enabled on the Quest

- [ ] **Step 6: Commit any fixes**

```bash
git add -A
git commit -m "fix: address integration test findings from Quest 3 hardware testing"
```

---

## Summary

This plan covers Phase 1 (Foundation) in 10 tasks across 3 chunks:

| Chunk | Tasks | What it delivers |
|-------|-------|-----------------|
| 1: Scaffolding & Protocol | Tasks 1-4 | Project structure, ADB message format, transport abstraction, RSA auth |
| 2: Connection & USB | Tasks 5-8 | ADB state machine, device model, IOKit USB transport, device manager |
| 3: Integration | Tasks 9-10 | Debug app, hardware testing |

After Phase 1, you'll have a working "Hello Quest" — the app detects a Quest 3 over USB, authenticates via ADB, and confirms the connection. This is the foundation everything else builds on.

Phase 2 (File Operations) plan will be written separately and builds on this foundation by adding the ADB sync service for browsing and transferring files.
