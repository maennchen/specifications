# Client Flows

This document describes how package manager clients interact with the Hex ecosystem, including API calls, caching behavior, and verification flows.

## Client Architecture Overview

```mermaid
C4Context
    title Client Library Architecture

    Person(dev, "Developer", "Runs build commands")

    Boundary(elixir_erlang, "Elixir/Erlang Clients") {
        System(mix, "Mix", "Elixir build tool")
        System(rebar3, "Rebar3", "Erlang build tool")
        System(hex_client, "Hex Client", "hexpm/hex - Mix integration")
        System(hex_core, "Hex Core", "hexpm/hex_core - Core library")
        System(hex_solver, "Hex Solver", "hexpm/hex_solver - Version resolver")
    }

    Boundary(gleam_client, "Gleam Client") {
        System(gleam, "Gleam", "Gleam build tool")
        System(hexpm_rust, "hexpm-rust", "gleam-lang/hexpm-rust - Rust Hex client")
    }

    System_Ext(hexpm, "hex.pm", "Package registry")

    Rel(dev, mix, "mix deps.get")
    Rel(dev, rebar3, "rebar3 compile")
    Rel(dev, gleam, "gleam build")

    Rel(mix, hex_client, "Uses")
    Rel(hex_client, hex_core, "Uses")
    Rel(hex_client, hex_solver, "Uses")
    Rel(rebar3, hex_core, "Uses directly")

    Rel(gleam, hexpm_rust, "Uses")

    Rel(hex_core, hexpm, "HTTPS", "Registry + tarballs")
    Rel(hexpm_rust, hexpm, "HTTPS", "Registry + tarballs")

    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="2")
```

---

## 1. Mix (Elixir) Dependency Installation

Command: `mix deps.get`

### Cache Locations

| Cache | Path | Contents |
|-------|------|----------|
| Registry | `~/.hex/cache.ets` | ETS file with package versions, deps, checksums |
| Packages | `~/.hex/packages/hexpm/{package}-{version}.tar` | Downloaded tarballs |
| Config | `~/.hex/hex.config` | API keys (encrypted), OAuth tokens |
| ETags | Inside `cache.ets` | `{:registry_etag, repo, package}` |

Environment variable `HEX_HOME` overrides the default `~/.hex` location.

### Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant Dev as Developer
    participant Mix as Mix
    participant Hex as Hex Client
    participant HexCore as Hex Core
    participant Solver as Hex Solver
    participant Cache as Local Cache<br/>~/.hex/
    participant CDN as Fastly CDN
    participant S3 as S3 Storage
    participant API as hex.pm API

    Dev->>Mix: mix deps.get
    Mix->>Hex: converge(deps, lock)

    Note over Hex,Cache: Registry Initialization
    Hex->>Cache: Open cache.ets
    Cache-->>Hex: ETS table reference

    Note over Hex,API: Authentication Check
    Hex->>Cache: Read hex.config
    alt Has OAuth Token
        Hex->>API: Check token expiry
        opt Token Expired
            Hex->>API: POST /api/oauth/token (refresh)
            API-->>Hex: New access token
            Hex->>Cache: Update hex.config
        end
    end

    Note over Hex,S3: Registry Prefetch
    Hex->>HexCore: get_package(repo, name, etag)

    loop For each dependency
        HexCore->>Cache: Get cached ETag
        Cache-->>HexCore: ETag or nil

        HexCore->>CDN: GET /packages/{name}<br/>If-None-Match: {etag}

        alt Cache Hit (304 Not Modified)
            CDN-->>HexCore: 304 Not Modified
            HexCore->>Cache: Use cached data
        else Cache Miss (200 OK)
            CDN->>S3: Fetch from origin
            S3-->>CDN: Signed protobuf (gzip)
            CDN-->>HexCore: 200 OK + ETag

            Note over HexCore: Verify RSA-SHA512 signature
            HexCore->>HexCore: gunzip → decode Signed protobuf
            HexCore->>HexCore: verify(payload, signature, public_key)

            alt Signature Invalid
                HexCore-->>Hex: {error, bad_signature}
                Hex-->>Mix: Abort with security error
            end

            HexCore->>Cache: Store package data + ETag
        end
    end

    Note over Hex,Solver: Dependency Resolution
    Hex->>Solver: resolve(requirements)
    Solver->>Cache: Read versions, deps from cache
    Solver->>Solver: Find optimal version set
    Solver-->>Hex: Resolved versions

    Note over Hex,S3: Package Download
    loop For each resolved package
        Hex->>Cache: Check ~/.hex/packages/{repo}/{name}-{version}.tar
        alt Cached & checksum matches
            Cache-->>Hex: Use cached tarball
        else Not cached or checksum mismatch
            Hex->>HexCore: get_tarball(repo, name, version)
            HexCore->>CDN: GET /tarballs/{name}-{version}.tar
            CDN->>S3: Fetch tarball
            S3-->>CDN: Package tarball
            CDN-->>HexCore: Tarball bytes

            Note over HexCore: Verify outer checksum (SHA-256)
            HexCore->>HexCore: SHA256(tarball) == registry.outer_checksum?

            alt Checksum Mismatch
                HexCore-->>Hex: {error, checksum_mismatch}
                Hex-->>Mix: Abort - possible tampering
            end

            HexCore-->>Hex: Tarball bytes
            Hex->>Cache: Save to packages/{repo}/
        end

        Hex->>Hex: Extract tarball to deps/{app}/
    end

    Hex->>Mix: Updated lock
    Mix->>Mix: Write mix.lock
    Mix-->>Dev: Dependencies fetched
```

### Error Handling

```mermaid
sequenceDiagram
    participant Hex as Hex Client
    participant Cache as Local Cache
    participant CDN as Fastly CDN

    Note over Hex,CDN: Registry Update Failure
    Hex->>CDN: GET /packages/{name}
    CDN-->>Hex: Network error / timeout

    Hex->>Cache: Check for cached registry
    alt Cache exists
        Cache-->>Hex: Cached registry data
        Hex->>Hex: ⚠️ Warn user: using cached registry
        Note over Hex: Continue with cached data
    else No cache
        Hex-->>Hex: ❌ Abort: cannot resolve dependencies
    end
```

---

## 2. Rebar3 (Erlang) Dependency Installation

Command: `rebar3 compile` (with hex plugin)

### Cache Locations

| Cache | Path | Contents |
|-------|------|----------|
| Registry | `~/.cache/rebar3/hex/hexpm/registry/` | Registry protobuf files |
| Packages | `~/.cache/rebar3/hex/hexpm/packages/` | Downloaded tarballs |
| Lock | `rebar.lock` | Resolved versions with checksums |

### Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant Dev as Developer
    participant Rebar as Rebar3
    participant HexCore as Hex Core<br/>(hex_core)
    participant Cache as Local Cache<br/>~/.cache/rebar3/hex/
    participant CDN as Fastly CDN
    participant S3 as S3 Storage

    Dev->>Rebar: rebar3 compile
    Rebar->>Rebar: Parse rebar.config

    Note over Rebar,S3: Registry Fetch
    Rebar->>HexCore: hex_repo:get_versions(Config)
    HexCore->>CDN: GET /versions
    CDN->>S3: Fetch (if not cached)
    S3-->>CDN: Signed protobuf (gzip)
    CDN-->>HexCore: Versions data

    Note over HexCore: Verify signature
    HexCore->>HexCore: gunzip → verify RSA-SHA512
    HexCore-->>Rebar: Package versions list

    loop For each dependency
        Rebar->>HexCore: hex_repo:get_package(Config, Name)
        HexCore->>Cache: Check ETag
        HexCore->>CDN: GET /packages/{name}<br/>If-None-Match: {etag}

        alt 304 Not Modified
            CDN-->>HexCore: Use cache
        else 200 OK
            CDN-->>HexCore: Package metadata
            HexCore->>HexCore: Verify signature
            HexCore->>Cache: Store + ETag
        end
        HexCore-->>Rebar: Package releases & deps
    end

    Note over Rebar: Dependency Resolution
    Rebar->>Rebar: Resolve version constraints

    Note over Rebar,S3: Download Packages
    loop For each resolved package
        Rebar->>Cache: Check packages/{name}-{version}.tar
        alt Not cached
            Rebar->>HexCore: hex_repo:get_tarball(Config, Name, Version)
            HexCore->>CDN: GET /tarballs/{name}-{version}.tar
            CDN-->>HexCore: Tarball

            Note over HexCore: Verify SHA-256 checksum
            HexCore->>HexCore: Unpack & verify checksums
            HexCore-->>Rebar: Package contents

            Rebar->>Cache: Cache tarball
        end
        Rebar->>Rebar: Extract to _build/
    end

    Rebar->>Rebar: Write rebar.lock
    Rebar->>Rebar: Compile dependencies
    Rebar-->>Dev: Build complete
```

---

## 3. Gleam Dependency Installation

Command: `gleam build`

### Cache Locations

| Cache | Path | Contents |
|-------|------|----------|
| Packages | `~/.cache/gleam/hex/hexpm/packages/` | Downloaded tarballs |
| Build | `./build/` | Compiled packages |
| Manifest | `manifest.toml` | Resolved versions with checksums |

### Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant Dev as Developer
    participant Gleam as Gleam
    participant HexRust as hexpm-rust
    participant CDN as Fastly CDN
    participant S3 as S3 Storage

    Dev->>Gleam: gleam build
    Gleam->>Gleam: Parse gleam.toml

    Note over Gleam,S3: Fetch Version Index
    Gleam->>HexRust: get_versions_request()
    HexRust->>CDN: GET /versions
    CDN->>S3: Fetch (if not cached)
    S3-->>CDN: Signed protobuf (gzip)
    CDN-->>HexRust: Compressed response

    Note over HexRust: Verify signature (ring crate)
    HexRust->>HexRust: gunzip → decode Signed protobuf
    HexRust->>HexRust: RSA-PKCS1-SHA512 verify
    HexRust-->>Gleam: All package versions

    loop For each dependency
        Gleam->>HexRust: get_package_request(name)
        HexRust->>CDN: GET /packages/{name}
        CDN-->>HexRust: Package metadata (signed)

        HexRust->>HexRust: Verify signature
        HexRust-->>Gleam: Releases & dependencies
    end

    Note over Gleam: Version Resolution
    Gleam->>Gleam: Resolve constraints (PubGrub algorithm)

    Note over Gleam,S3: Download Packages
    loop For each resolved package
        Gleam->>HexRust: get_tarball_request(name, version)
        HexRust->>CDN: GET /tarballs/{name}-{version}.tar
        CDN-->>HexRust: Tarball bytes

        Note over HexRust: Verify SHA-256 checksum
        HexRust->>HexRust: SHA256(tarball) == expected?

        alt Checksum Mismatch
            HexRust-->>Gleam: IncorrectChecksum error
            Gleam-->>Dev: ❌ Abort - integrity failure
        end

        HexRust-->>Gleam: Verified tarball
        Gleam->>Gleam: Extract to build/packages/
    end

    Gleam->>Gleam: Write manifest.toml
    Gleam->>Gleam: Compile Gleam + Erlang
    Gleam-->>Dev: Build complete
```

---

## 4. Package Publishing

Command: `mix hex.publish` or `rebar3 hex publish`

### Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant Dev as Developer
    participant Client as Hex Client
    participant Local as Local Filesystem
    participant API as hex.pm API
    participant S3 as S3 Storage
    participant Worker as Registry Worker

    Dev->>Client: mix hex.publish / rebar3 hex publish

    Note over Client,Local: Build Tarball
    Client->>Local: Read source files
    Client->>Client: Generate metadata.config
    Client->>Client: Create contents.tar.gz
    Client->>Client: Calculate inner checksum (SHA-256)
    Client->>Client: Write CHECKSUM file
    Client->>Client: Create outer tarball

    Note over Client: Tarball structure:<br/>VERSION (3)<br/>metadata.config<br/>contents.tar.gz<br/>CHECKSUM

    Note over Client,API: Authentication
    Client->>Client: Read API key from ~/.hex/hex.config
    alt No API key
        Client->>Dev: Prompt for credentials
        Dev-->>Client: Username + password
        Client->>API: POST /api/keys (Basic Auth)
        API-->>Client: API key
        Client->>Local: Save to hex.config
    end

    Note over Client,S3: Upload Package
    Client->>API: POST /api/publish<br/>Authorization: {api_key}<br/>Body: tarball

    API->>API: Validate tarball structure
    API->>API: Verify checksums
    API->>API: Check package ownership
    API->>API: Validate version (semver)

    alt Validation Failed
        API-->>Client: 422 Unprocessable Entity
        Client-->>Dev: ❌ Publish failed: {reason}
    end

    API->>S3: Store tarball at /tarballs/{name}-{version}.tar
    S3-->>API: Stored

    API->>API: Insert release into database
    API->>Worker: Trigger registry rebuild

    Worker->>Worker: Rebuild /names protobuf
    Worker->>Worker: Rebuild /versions protobuf
    Worker->>Worker: Rebuild /packages/{name} protobuf
    Worker->>Worker: Sign all with RSA-SHA512
    Worker->>Worker: Gzip compress
    Worker->>S3: Upload registry files

    API-->>Client: 201 Created
    Client-->>Dev: ✓ Package published

    Note over Dev: Optional: Publish docs
    Dev->>Client: mix hex.docs (automatic)
    Client->>Client: Generate documentation
    Client->>API: POST /api/packages/{name}/releases/{version}/docs<br/>Body: docs.tar.gz
    API->>S3: Store at hexdocs path
    API-->>Client: 201 Created
```

---

## 5. Private Package Flow

For organizations with private packages on hex.pm.

### Endpoints

Private packages use repository-scoped endpoints:

| Endpoint | Description |
|----------|-------------|
| `/repos/{org}/names` | Package names in organization |
| `/repos/{org}/versions` | Package versions in organization |
| `/repos/{org}/packages/{name}` | Package metadata |
| `/repos/{org}/tarballs/{name}-{version}.tar` | Package tarball |

### Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant Dev as Developer
    participant Client as Hex Client
    participant Cache as Local Cache
    participant CDN as Fastly CDN
    participant API as hex.pm

    Note over Dev,API: Private Package Fetch

    Dev->>Client: mix deps.get

    Note over Client: Check repository config
    Client->>Client: Read repo config from mix.exs
    Client->>Client: Identify private repos (org: "mycompany")

    Note over Client,API: Authenticate for private repo
    Client->>Cache: Read API key for "mycompany" org
    Cache-->>Client: API key

    Client->>CDN: GET /repos/mycompany/packages/{name}<br/>Authorization: {api_key}

    Note over CDN: Private packages NOT cached<br/>(per-user authorization)
    CDN->>API: Forward with auth
    API->>API: Verify API key has repo access
    API-->>CDN: 200 OK + Package data (signed)
    CDN-->>Client: Package metadata

    Client->>Client: Verify signature

    Note over Client,API: Download private tarball
    Client->>CDN: GET /repos/mycompany/tarballs/{name}-{version}.tar<br/>Authorization: {api_key}
    CDN->>API: Forward with auth
    API-->>CDN: Tarball
    CDN-->>Client: Tarball

    Client->>Client: Verify checksum
    Client->>Cache: Store in ~/.hex/packages/mycompany/

    Client-->>Dev: Private dependency fetched
```

### Security: Untrusted Mirrors

```mermaid
sequenceDiagram
    participant Client as Hex Client
    participant Mirror as Untrusted Mirror
    participant Hexpm as hex.pm

    Note over Client,Hexpm: ⚠️ CRITICAL: Never send credentials to untrusted mirrors

    Client->>Client: Check mirror configuration
    Client->>Client: mirror = "https://mirror.example.com"

    alt Public Package
        Client->>Mirror: GET /packages/{name}<br/>(NO Authorization header)
        Mirror-->>Client: Package metadata
        Client->>Client: Verify signature with hex.pm public key
    else Private Package
        Note over Client: Private packages MUST use trusted endpoint
        Client->>Hexpm: GET /repos/{org}/packages/{name}<br/>Authorization: {api_key}
        Hexpm-->>Client: Package metadata
    end
```

---

## Verification Summary

All clients perform these verification steps:

### Registry Verification

1. **Decompress** - gunzip the response
2. **Decode** - Parse `Signed` protobuf wrapper
3. **Verify Signature** - RSA-PKCS1-SHA512 of payload using hex.pm public key
4. **Verify Repository** - Ensure `repository` field matches expected repo name
5. **Parse Payload** - Decode inner protobuf (Names, Versions, or Package)

### Tarball Verification

1. **Download** - Fetch tarball from `/tarballs/{name}-{version}.tar`
2. **Outer Checksum** - `SHA256(entire tarball)` must match registry's `outer_checksum`
3. **Extract** - Unpack tar to get VERSION, metadata.config, contents.tar.gz, CHECKSUM
4. **Inner Checksum** - Verify CHECKSUM file matches `SHA256(VERSION + metadata.config + contents.tar.gz)`

### Public Key

The hex.pm public key is:
- Distributed with client libraries (hardcoded)
- Available at `https://hex.pm/docs/public_keys`
- 2048-bit RSA key used for RSA-PKCS1-SHA512 signatures
