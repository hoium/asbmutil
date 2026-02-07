# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

`asbmutil` is a Swift 6 command-line tool for interacting with the Apple School & Business Manager (AxM) API. It enables device management operations including listing devices, assigning/unassigning MDM servers, and querying AppleCare coverage for organizations managing Apple devices at scale.

## Technology Stack

- **Language:** Swift 6 with StrictConcurrency enabled
- **Platform:** macOS 14+
- **Dependencies:**
  - `swift-argument-parser` (1.3.0+) for CLI interface
- **Runtime:** Pure Swift binary with no external runtime dependencies
- **Authentication:** OAuth 2 client assertion (JWT) with Apple's Account API
- **Storage:** macOS Keychain for secure credential management

## Build & Development Commands

### Build

```bash
# Debug build
swift build

# Release build (recommended for distribution)
swift build -c release

# Binary location
.build/release/asbmutil
```

### Run

```bash
# Run from build directory
cd .build/release
./asbmutil <command>

# Or directly with swift run (debug mode)
swift run asbmutil <command>
```

### Testing

**Note:** This repository currently has no test suite. When adding tests, use:

```bash
swift test
```

## Architecture

### Command Structure

The CLI uses `ArgumentParser` with a hierarchical command structure:

- **Main Entry:** `Entry.swift` defines `ASBMUtil` as the root `AsyncParsableCommand`
- **Subcommands:**
  - Device operations: `list-devices`, `assign`, `unassign`, `get-assigned-mdm`, `get-applecare`
  - MDM operations: `list-mdm-servers`, `batch-status`
  - Config operations: `config set`, `config show`, `config clear`, `config list-profiles`, `config set-profile`, `config show-profile`

### Core Components

#### APIClient (Actor)

**File:** `APIClient.swift`

The `APIClient` actor is the central HTTP client for all AxM API communication:

- **Concurrency:** Actor isolation ensures thread-safe token management and request coordination
- **Authentication:** Automatically handles OAuth 2 token fetching and refresh using JWT client assertions
- **Retry Logic:** Implements exponential backoff with jitter for rate limiting (429) and server errors (5xx)
- **Pagination:** Handles cursor-based pagination for device listing with configurable limits
- **JWT Generation:** Creates ES256-signed JWTs using CryptoKit (P256 signing keys)
- **PEM Key Support:** Handles both PKCS#8 and SEC-1 (EC) private key formats via `/usr/bin/openssl` conversion

**Key Methods:**

- `listDevices()` - Paginated device fetching with optional limits
- `createDeviceActivity()` - Assign/unassign devices to MDM servers
- `listMdmServers()` - Fetch available MDM servers
- `getAppleCareCoverage()` - Query AppleCare coverage (API 1.3)
- `send()` - Generic HTTP request method with automatic token refresh

#### Credential Management & Profiles

**Files:** `Keychain.swift`, `Credentials.swift`, `Creds.swift`, `ConfigCommand.swift`

**Profile System:**

- Supports multiple AxM instances via named profiles (e.g., "school-district-2", "business-unit-3")
- Credentials stored securely in macOS Keychain under service `com.github.rodchristiansen.asbmutil`
- Profile metadata tracked separately (client ID, scope, creation date)
- "default" profile used when none specified
- Current profile setting persisted across sessions

**Profile Workflow:**

1. User stores credentials: `config set --profile <name> --client-id <id> --key-id <kid> --pem-path <path>`
2. Keychain saves `KCBlob` (clientId, keyId, privateKey PEM, teamId)
3. Profile added to profiles list with metadata
4. Profile set as current if first one or explicitly requested
5. All commands support `--profile` flag to override current profile

**Scope Detection:**

- Automatically determines `school.api` vs `business.api` based on client ID prefix:
  - `SCHOOLAPI.*` → `https://api-school.apple.com/`
  - `BUSINESSAPI.*` → `https://api-business.apple.com/`

#### Device Models & API Responses

**File:** `Device.swift`

**Complex Type Handling:**

- `StringOrArray` enum handles fields that Apple's API returns as either `String` or `[String]` (e.g., `imei` for dual-SIM devices)
- `DeviceAttributes` includes all AxM API v1.4 fields including:
  - MAC addresses (Wi-Fi, Bluetooth, Ethernet for macOS)
  - Network identifiers (IMEI, MEID, EID)
  - Product details (family, type, model, capacity, color)
  - Order/purchase information
  - Status and timestamps

**Key Structures:**

- `OrgDevicesResponse` - Paginated device list response with cursor metadata
- `MdmServersResponse` - MDM server list response
- `AppleCareResponse` - AppleCare coverage response (API 1.3)
- `ActivityDetails` - Device assignment/unassignment operation details

#### Endpoints & HTTP

**Files:** `Endpoints.swift`, `HTTP.swift`

- `Endpoints` enum provides type-safe API paths for v1 endpoints
- Base URL determined by scope (`school.api` vs `business.api`)
- `Request<T>` generic struct wraps HTTP requests with method, path, scope, and optional body
- Proxy disabled for all requests to avoid PAC/proxy interference

### Data Flow Example: Assigning Devices

1. User runs: `./asbmutil assign --serials ABC123,DEF456 --mdm "Intune" --profile "school-1"`
2. `Assign.run()` in `Commands.swift`:
   - Loads credentials via `Creds.load(profileName: "school-1")`
   - Resolves profile to `Credentials` (clientId, keyId, privateKey, scope)
3. Creates `APIClient` actor:
   - Initializer calls `fetchToken()` to get OAuth 2 bearer token
   - Generates JWT signed with P256 private key from Keychain
   - POSTs to `https://account.apple.com/auth/oauth2/token`
4. Looks up MDM server ID by name via `getMdmServerIdByName()`
5. Calls `createDeviceActivity()`:
   - Builds JSON:API compliant request body
   - POSTs to `/v1/orgDeviceActivities`
   - Returns activity ID for status tracking
6. Outputs JSON with activity details including server name, device count, serials

## CSV File Format

For bulk operations (`--csv-file`), the tool reads serial numbers from the **first column only**:

```csv
P8R2K47NF5X9,MacBook Air,2023
Q7M5V83WH4L2,MacBook Pro,2022
```

Additional columns are ignored. Helper function `readSerialsFromCSV()` in `Commands.swift` handles parsing.

## API Versioning

The tool supports AxM API features up to v1.4:

- **v1.2:** Wi-Fi/Bluetooth MAC addresses for iOS, iPadOS, tvOS, visionOS
- **v1.3:** AppleCare coverage lookup
- **v1.4:** Built-in Ethernet MAC address for macOS

When adding new API version features, update `DeviceAttributes` in `Device.swift` and add corresponding endpoints in `Endpoints.swift`.

## Error Handling

- **RuntimeError:** Custom error type in `RuntimeError.swift` for domain-specific errors
- **ValidationError:** ArgumentParser validation failures
- **HTTP Errors:** APIClient provides diagnostic output to stderr before throwing
- **Retry Logic:** Automatic retry for 429, 5xx, and 408 status codes
- **Keychain Errors:** OSStatus codes from Security framework

## Code Signing & Distribution

When preparing releases, sign and notarize the binary:

```bash
# Sign
codesign --force --sign "Developer ID Application: Your Name (TEAM_ID)" .build/release/asbmutil

# Verify
codesign --verify --verbose .build/release/asbmutil

# Notarize
zip asbmutil.zip .build/release/asbmutil
xcrun notarytool submit asbmutil.zip --keychain-profile "notarytool-profile" --wait

# Staple
xcrun stapler staple .build/release/asbmutil
```

## Commit & Pull Request Guidelines

### Commit Message Format

```text
<type>: <description>

[optional body]

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
```

**Commit Types:**

- `feat`: New feature or command (e.g., "feat: Add support for device transfer operations")
- `fix`: Bug fix (e.g., "fix: Handle missing AppleCare coverage gracefully")
- `refactor`: Code restructuring without behavior change (e.g., "refactor: Extract pagination logic into separate method")
- `chore`: Maintenance tasks (e.g., "chore: Update swift-argument-parser to 1.4.0")
- `docs`: Documentation changes (e.g., "docs: Update README with API 1.4 features")

### Pull Request Format

**Title:** `<type>: <short description under 70 chars>`

**Description:**

```markdown
## Summary
- Bullet point summary of changes
- What problem this solves
- Any breaking changes

## Test Plan
- [ ] Tested on macOS 14.x with school.api
- [ ] Tested on macOS 14.x with business.api
- [ ] Verified Keychain profile management
- [ ] Tested with large device inventories (>1000 devices)

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

## Security Considerations

- **Never commit** PEM private keys, client IDs, or key IDs
- Credentials stored in macOS Keychain only (not in files)
- JWT tokens have 20-minute expiration (`exp: now + 1_200`)
- Token refresh handled automatically before expiration
- Profile system isolates credentials between ABM instances

## Default Branch

The default branch for this repository is `main`.
