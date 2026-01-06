# Hex Infrastructure

This document provides architectural diagrams for the Hex package manager ecosystem using the C4 model.

## Ecosystem Inventory

### Hex Services

| Name | Description | Production | Staging | Repository |
|------|-------------|------------|---------|------------|
| Hex Registry | Main registry, web UI & API | hex.pm | staging.hex.pm | [hexpm/hexpm](https://github.com/hexpm/hexpm) |
| Hex Operations | Terraform, Config, Fastly Compute | - | - | [hexpm/hexpm-ops](https://github.com/hexpm/hexpm-ops) (🔒) |
| Hex Docs | Documentation hosting | hexdocs.pm | staging.hexdocs.pm | [hexpm/hexdocs](https://github.com/hexpm/hexdocs), [hexpm/hexdocs-search](https://github.com/hexpm/hexdocs-search) |
| Hex Preview | Package preview | preview.hex.pm | - | [hexpm/preview](https://github.com/hexpm/preview) |
| Hex Diff | Package diff viewer | diff.hex.pm | - | [hexpm/diff](https://github.com/hexpm/diff) |

### Client Libraries

| Name | Description | Repository |
|------|-------------|------------|
| Hex | Elixir Hex client | [hexpm/hex](https://github.com/hexpm/hex) |
| Hex Core | Core library for Elixir/Erlang clients | [hexpm/hex_core](https://github.com/hexpm/hex_core) |
| Hex Solver | Version constraint resolver | [hexpm/hex_solver](https://github.com/hexpm/hex_solver) |
| hexpm-rust | Rust Hex client (used by Gleam) | [gleam-lang/hexpm-rust](https://github.com/gleam-lang/hexpm-rust) |

### Build Tools

| Name | Description | Repository |
|------|-------------|------------|
| Mix | Elixir build tool | [elixir-lang/elixir](https://github.com/elixir-lang/elixir) (lib/mix) |
| Rebar3 | Erlang build tool | [erlang/rebar3](https://github.com/erlang/rebar3) |
| Gleam | Gleam language & build tool | [gleam-lang/gleam](https://github.com/gleam-lang/gleam) |

---

## C4 Level 1: System Context

Shows the Hex ecosystem and its relationships with users and external systems.

```mermaid
C4Context
    title System Context Diagram for Hex Package Manager

    Person(dev, "Developer", "Elixir/Erlang/Gleam developer using mix, rebar3, or gleam")
    Person(maintainer, "Package Maintainer", "Publishes and manages packages")
    Person(ci, "CI/CD System", "Automated build systems fetching packages")

    System(hex, "Hex.pm", "Package registry providing packages, documentation, and API")

    System_Ext(github, "GitHub", "OAuth authentication and source code hosting")
    System_Ext(email, "Email Service", "Sends notifications and password resets")
    System_Ext(fastly, "Fastly CDN", "Global content delivery and edge compute")

    Rel(dev, hex, "Fetches packages", "HTTPS")
    Rel(maintainer, hex, "Publishes packages", "HTTPS + API Key")
    Rel(ci, hex, "Fetches packages", "HTTPS")

    Rel(hex, github, "OAuth authentication")
    Rel(hex, email, "Sends notifications")
    Rel(hex, fastly, "Serves content via")

    Rel(fastly, dev, "Delivers registry + tarballs", "HTTPS cached")
    Rel(fastly, ci, "Delivers registry + tarballs", "HTTPS cached")

    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```

### Context Description

**Users:**
- **Developers** fetch packages during development and CI builds
- **Package Maintainers** publish and manage packages via the web UI or CLI
- **CI/CD Systems** fetch packages during automated builds

**External Systems:**
- **GitHub** provides OAuth authentication and hosts package source code
- **Email Service** sends password resets, ownership notifications, etc.
- **Fastly CDN** caches and delivers registry files and package tarballs globally

---

## C4 Level 2: Container Diagram

Shows the internal services and data stores within the Hex ecosystem.

```mermaid
C4Container
    title Container Diagram for Hex Package Manager

    Person(dev, "Developer", "Uses mix, rebar3, or gleam")
    Person(maintainer, "Package Maintainer", "Publishes packages")

    System_Boundary(hex, "Hex Ecosystem") {
        Container(fastly, "Fastly CDN", "Compute@Edge", "Caches content, routes requests, runs edge logic")

        Container(hexpm_web, "hex.pm", "Phoenix/Elixir", "Web UI, HTTP API, package management")
        Container(hexdocs, "hexdocs.pm", "Phoenix/Elixir", "Documentation hosting and search")
        Container(preview, "preview.hex.pm", "Phoenix/Elixir", "Package preview before publishing")
        Container(diff, "diff.hex.pm", "Phoenix/Elixir", "Visual diff between package versions")

        ContainerDb(postgres, "PostgreSQL", "Database", "Users, packages, releases, API keys, organizations")
        ContainerDb(s3, "Object Storage", "S3", "Tarballs, registry files, documentation")

        Container(registry_worker, "Registry Builder", "Elixir Worker", "Rebuilds registry after publishes")
        Container(notification_worker, "Notification Worker", "Elixir Worker", "Sends emails and webhooks")
    }

    System_Ext(email, "Email Service", "Sends notifications")

    Rel(dev, fastly, "Fetches packages", "HTTPS")
    Rel(maintainer, fastly, "Publishes packages", "HTTPS")

    Rel(fastly, s3, "Registry files", "/names, /versions, /packages/*")
    Rel(fastly, s3, "Tarballs", "/tarballs/*")
    Rel(fastly, hexpm_web, "API requests", "/api/*")
    Rel(fastly, hexdocs, "Documentation", "HTTPS")

    Rel(hexpm_web, postgres, "Reads/writes", "PostgreSQL")
    Rel(hexpm_web, s3, "Stores packages", "S3 API")
    Rel(hexpm_web, registry_worker, "Triggers rebuild")
    Rel(hexpm_web, notification_worker, "Queues notifications")

    Rel(hexdocs, s3, "Reads docs", "S3 API")
    Rel(preview, hexpm_web, "Fetches package data", "Internal API")
    Rel(diff, hexpm_web, "Fetches package versions", "Internal API")

    Rel(registry_worker, s3, "Writes registry", "S3 API")
    Rel(notification_worker, email, "Sends email", "SMTP")

    UpdateLayoutConfig($c4ShapeInRow="4", $c4BoundaryInRow="1")
```

### Container Descriptions

**CDN Layer:**
- **Fastly CDN + Compute@Edge**: Handles all incoming traffic, caches registry files and tarballs, routes requests to appropriate origins. Compute@Edge runs custom logic from hexpm-ops.

**Hex Services:**
- **hex.pm**: Main Phoenix application providing web UI, HTTP API (`/api/*`), and repository endpoints. Handles authentication, package publishing, user management, and organization management.
- **hexdocs.pm**: Serves generated documentation for packages. Integrates with hexdocs-search for documentation search.
- **preview.hex.pm**: Allows maintainers to preview how a package will appear before publishing.
- **diff.hex.pm**: Provides visual diffs between package versions for security review.

**Data Stores:**
- **PostgreSQL**: Primary database storing users, packages, releases, dependencies, API keys, organizations, and audit logs.
- **Object Storage (S3)**: Stores package tarballs, documentation tarballs, and pre-built registry files (protobuf + gzip + signed).

**Background Workers:**
- **Registry Builder**: Triggered after package publish/update to rebuild registry files (`/names`, `/versions`, `/packages/*`).
- **Notification Worker**: Sends email notifications for password resets, ownership changes, security advisories.

---

## C4 Deployment Diagram

Shows the infrastructure deployment topology.

```mermaid
C4Deployment
    title Deployment Diagram for Hex Package Manager

    Deployment_Node(internet, "Internet", "") {
        Deployment_Node(client_node, "Client Machine", "") {
            Container(client, "Build Tool", "mix/rebar3/gleam", "Fetches packages and registry")
        }
        Deployment_Node(browser_node, "Browser", "") {
            Container(browser, "Web Browser", "", "Access web UI")
        }
    }

    Deployment_Node(fastly_edge, "Fastly Edge", "Global POPs") {
        Container(edge, "Edge Cache + Compute@Edge", "Fastly", "Caches registry and tarballs, routes requests")
    }

    Deployment_Node(production, "Production Environment", "") {
        Deployment_Node(web_tier, "Web Tier", "") {
            Container(hex_prod, "hex.pm", "Phoenix/Elixir", "Main application")
            Container(hexdocs_prod, "hexdocs.pm", "Phoenix/Elixir", "Documentation")
            Container(preview_prod, "preview.hex.pm", "Phoenix/Elixir", "Preview")
            Container(diff_prod, "diff.hex.pm", "Phoenix/Elixir", "Diff viewer")
        }
        Deployment_Node(data_tier, "Data Tier", "") {
            ContainerDb(pg_prod, "PostgreSQL", "Primary + Replica", "Application data")
            ContainerDb(s3_prod, "S3 Bucket", "Object Storage", "Tarballs, registry, docs")
        }
    }

    Deployment_Node(staging, "Staging Environment", "") {
        Container(hex_staging, "staging.hex.pm", "Phoenix/Elixir", "Staging application")
        Container(hexdocs_staging, "staging.hexdocs.pm", "Phoenix/Elixir", "Staging docs")
        ContainerDb(pg_staging, "PostgreSQL", "Database", "Staging data")
        ContainerDb(s3_staging, "S3 Bucket", "Object Storage", "Staging artifacts")
    }

    Deployment_Node(mirrors, "Mirror Infrastructure", "") {
        Container(trusted_mirror, "Trusted Mirror", "Org-controlled", "Full authentication support")
        Container(untrusted_mirror, "Untrusted Mirror", "Community", "Public packages only - NO credentials")
    }

    Rel(client, edge, "HTTPS")
    Rel(browser, edge, "HTTPS")

    Rel(edge, hex_prod, "Cache MISS", "/api/*")
    Rel(edge, s3_prod, "Cache MISS", "Registry + tarballs")

    Rel(hex_prod, pg_prod, "PostgreSQL")
    Rel(hex_prod, s3_prod, "S3 API")

    Rel(hexdocs_prod, s3_prod, "S3 API")

    Rel(hex_staging, pg_staging, "PostgreSQL")
    Rel(hex_staging, s3_staging, "S3 API")

    Rel(client, trusted_mirror, "Alternative path", "With auth")
    Rel(client, untrusted_mirror, "Alternative path", "NO auth - public only")

    Rel(trusted_mirror, hex_prod, "Proxies to origin")
    Rel(untrusted_mirror, s3_prod, "Public packages only")
```

### Deployment Notes

**CDN Strategy:**
- All traffic flows through Fastly edge locations globally
- Registry files (`/names`, `/versions`, `/packages/*`) are heavily cached
- Tarballs are cached with long TTLs (immutable content)
- API requests (`/api/*`) pass through to origin

**Trust Boundaries:**
- **Trusted path**: Client → Fastly → hex.pm (credentials safe)
- **Untrusted mirrors**: Clients MUST NOT send credentials; only fetch public packages

**Staging:**
- `staging.hex.pm` and `staging.hexdocs.pm` for testing
- Separate database and storage from production

**High Availability:**
- S3 provides durability for package artifacts
- Fastly provides global redundancy

---

## Communication Protocols

| Path | Protocol | Format | Authentication |
|------|----------|--------|----------------|
| Client → Registry files | HTTPS | Protobuf + gzip + RSA-SHA512 signed | None (public) or API key (private) |
| Client → Tarballs | HTTPS | tar (VERSION, metadata.config, contents.tar.gz, CHECKSUM) | None (public) or API key (private) |
| Client → API | HTTPS | JSON | API key (Bearer token) |
| Browser → Web | HTTPS | HTML | Session cookie |
| hex.pm → PostgreSQL | TCP | PostgreSQL protocol | Connection credentials |
| hex.pm → S3 | HTTPS | S3 API | IAM credentials |

---

## Trust Boundaries

```mermaid
C4Context
    title Trust Boundary Diagram for Hex Package Manager

    Boundary(trusted, "Trusted Zone") {
        System(hexpm, "hex.pm", "Main application")
        System(fastly, "Fastly CDN", "Edge delivery")
        SystemDb(s3, "S3 Storage", "Package artifacts")
        SystemDb(pg, "PostgreSQL", "Application data")
    }

    Boundary(semi_trusted, "Semi-Trusted Zone") {
        System(trusted_mirror, "Trusted Mirrors", "Organization-controlled mirrors")
    }

    Boundary(untrusted, "Untrusted Zone") {
        Person(client, "Client", "Build tools: mix, rebar3, gleam")
        System(untrusted_mirror, "Untrusted Mirrors", "Community mirrors")
    }

    Rel(client, fastly, "Credentials OK", "HTTPS + API Key")
    Rel(client, trusted_mirror, "Credentials OK", "HTTPS + API Key")
    Rel(client, untrusted_mirror, "NO CREDENTIALS", "HTTPS only")

    Rel(fastly, hexpm, "Internal")
    Rel(trusted_mirror, hexpm, "Proxies requests")
    Rel(untrusted_mirror, s3, "Public packages only")

    UpdateLayoutConfig($c4ShapeInRow="4", $c4BoundaryInRow="3")
```

**Key Security Principle:** Clients must NEVER send API keys or authentication tokens to untrusted mirrors. Even if configured to use a mirror, authentication must be disabled for untrusted endpoints.
