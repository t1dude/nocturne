# Nocturne

A modern, high-performance diabetes management platform built with .NET 10. Nocturne is a complete rewrite of the Nightscout API with full feature parity, providing native C# implementations of all endpoints with optimized performance and modern cloud-native architecture.

## What is Nocturne?

Nocturne is a comprehensive diabetes data platform that provides:

- **Complete Nightscout API Implementation** - All Nightscout endpoints natively implemented in C# with full compatibility
- **Data Connectors** - Native integration with major diabetes platforms (Dexcom, Glooko, LibreLinkUp, MiniMed CareLink, MyFitnessPal, Nightscout)
- **Real-time Updates** - WebSocket/SignalR support for live glucose readings and alerts
- **Advanced Analytics** - Comprehensive glucose statistics, time-in-range calculations, and reports
- **Cloud-Native** - Built on Aspire for seamless local development and cloud deployment

## Architecture

```
Nocturne/
├── src/
│   ├── API/                        # REST API (Nightscout-compatible)
│   ├── Aspire/                     # Aspire orchestration
│   ├── Connectors/                 # Data source integrations (Dexcom, Libre, etc.)
│   ├── Core/                       # Domain models, interfaces, and constants
│   ├── Desktop/                    # Desktop application
│   ├── Infrastructure/             # EF Core data access, caching, security
│   ├── Portal/                     # Marketing website
│   ├── Services/                   # Background services
│   ├── Tools/                      # CLI tools and MCP server
│   ├── Web/                        # pnpm monorepo (SvelteKit frontend, bot, bridge)
│   └── Widgets/                    # Embeddable widgets
└── tests/                          # Comprehensive test suite
```

## Key Features

- **Full Nightscout API Parity** - All v1, v2, and v3 endpoints
- **High Performance** - Optimized queries with PostgreSQL
- **Authentication** - JWT-based auth with API_SECRET support
- **Real-time** - SignalR hubs for live data streaming
- **Data Connectors** - Dexcom Share, Glooko, LibreLinkUp, MiniMed CareLink, MyFitnessPal, Nightscout, and MyLife
- **PostgreSQL** - Modern relational database with EF Core migrations
- **Observability** - OpenTelemetry integration for monitoring (Soon)
- **Containerized** - Docker support for all services

## Quick Start with Aspire (Development)

### Prerequisites

- [.NET 10.0 SDK](https://dotnet.microsoft.com/download/dotnet/10.0) or later
- [Docker Desktop](https://www.docker.com/products/docker-desktop)
- [Node.js 24+](https://nodejs.org/)
- [pnpm 9+](https://pnpm.io/)

Aspire orchestrates all services with a single command:

```bash
aspire start
```

Aspire will automatically:

- Start PostgreSQL in a container
- Run database migrations
- Start the Nocturne API and SvelteKit frontend
- Launch any configured data connectors
- Set up service discovery, health checks, and a YARP gateway

Once running, open the Aspire dashboard link from the console output to see all services. Access the app at `https://localhost:1612`.

## Multitenancy (Custom Local Domain)

By default, Aspire serves the app at `https://localhost:1612`. This works for single-tenant development but **WebAuthn passkeys fail on tenant subdomains** because browsers reject `localhost` as a passkey Relying Party ID for subdomain origins.

To test multitenancy with passkeys locally:

**1. Install mkcert**

```bash
# Windows
winget install FiloSottile.mkcert

# macOS
brew install mkcert

# Linux — use your distro's package manager
```

**2. Set the custom domain**

```bash
cd src/Aspire/Nocturne.Aspire.Host
dotnet user-secrets set "LocalDev:Domain" "nocturne.test"
```

**3. Add hosts file entries**

Add lines to your hosts file (`C:\Windows\System32\drivers\etc\hosts` on Windows, `/etc/hosts` on macOS/Linux):

```
127.0.0.1  nocturne.test
127.0.0.1  demo.nocturne.test
127.0.0.1  riley.nocturne.test
```

Add one line per tenant slug you want to use. Hosts files don't support wildcards.

**4. Start Aspire**

```bash
aspire start
```

Aspire will automatically generate a wildcard TLS certificate for `*.nocturne.test`, install the mkcert CA into your system trust store, and configure the YARP gateway to use it. Access the app at `https://nocturne.test`.

## Production Deployment (Docker Compose)

The easiest way to deploy Nocturne is with the production Docker Compose bundle. Each [GitHub Release](https://github.com/nightscout/nocturne/releases) includes ready-to-use artifacts, or you can generate them locally.

### Using a release

Download `docker-compose.yaml` and `.env.example` from the [latest release](https://github.com/nightscout/nocturne/releases).

```bash
# 1. Copy the env template and fill in your passwords and domain
cp .env.example .env

# 2. Start Nocturne
docker compose up -d
```

The production compose includes [Watchtower](https://github.com/nicholas-fedor/watchtower) for automatic container updates (checks daily), and omits the Aspire dashboard and Scalar API explorer. Watchtower will automatically pull new images as they are published — no manual updates needed.

### Generating locally

If you have the .NET 10 SDK and Aspire CLI installed, you can generate the production bundle from source:

```bash
dotnet run scripts/publish-release.cs              # outputs to ./release-output
dotnet run scripts/publish-release.cs ./deploy     # or specify a directory
```

### PostgreSQL Roles

Nocturne uses three separate PostgreSQL roles for defense in depth. All three have `NOBYPASSRLS` so they obey Row Level Security policies, even when the database has no superuser connected.

| Role                    | Purpose                                                                                | Privileges                                                                                                     |
| ----------------------- | -------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| **`nocturne_migrator`** | Runs EF Core migrations (schema DDL). Owns the database and `public` schema.           | `CREATE`, `ALTER`, `DROP` on tables. Cannot bypass RLS.                                                        |
| **`nocturne_app`**      | Runtime connection pool for the .NET API. Owns nothing.                                | `SELECT`, `INSERT`, `UPDATE`, `DELETE` on migrator-created tables. Cannot bypass RLS.                          |
| **`nocturne_web`**      | SvelteKit bot framework (chat state storage). Owns only its own `chat_state_*` tables. | `CREATE` on `public` schema (for its own tables only). No access to Nocturne tenant tables. Cannot bypass RLS. |

The bootstrap user (`POSTGRES_USER`) is only used for initial container setup. After `container-init/00-init.sh` runs, all application traffic flows through the three roles above. Passwords are set via environment variables in `.env`.

For bring-your-own PostgreSQL (not using the bundled container), run `docs/postgres/bootstrap-roles.sql` once as a superuser. See the comments in that file for details.

## Production Deployment (Cosmos)

[Cosmos](https://cosmos-cloud.io/) is a self-hosted application server and reverse proxy. It handles TLS, authentication, and auto-updates, so the Nocturne stack does not need an external reverse proxy of its own.

### Prerequisites

- A running Cosmos instance (v0.9.0 or later)
- Docker available on the host

### Deploying from the marketplace

If Nocturne is listed in a Cosmos marketplace you have configured, open **Apps → Market**, search for **Nocturne**, and click **Install**. Cosmos will prompt for a service name and generate all passwords automatically — skip to [Post-deployment configuration](#post-deployment-configuration) below.

### Deploying from the compose file

1. In the Cosmos UI, go to **Apps → Create App → Docker Compose**.
2. Paste the contents of [`deploy/cosmos/cosmos-compose.json`](deploy/cosmos/cosmos-compose.json) into the editor.
3. Choose a service name (e.g. `nocturne`). Cosmos replaces `{ServiceName}` throughout the template.
4. Click **Install**. Cosmos generates random passwords for all `{Passwords.N}` placeholders and starts the stack.

The following containers are created:

| Container | Role |
|---|---|
| `<name>` | YARP gateway — Cosmos routes external traffic here |
| `<name>-api` | Nocturne .NET API |
| `<name>-web` | SvelteKit frontend |
| `<name>-postgres` | PostgreSQL 17 |

### Bootstrap PostgreSQL roles

Cosmos's compose format does not support the Docker `configs` mechanism used to seed the database on first start, so the three PostgreSQL roles must be created manually once after the first deployment. Find the generated passwords in the container environment variables via the Cosmos UI, then run:

```bash
docker exec -i <name>-postgres psql -U nocturne -d nocturne <<'SQL'
CREATE ROLE nocturne_migrator LOGIN NOSUPERUSER NOBYPASSRLS NOCREATEDB NOCREATEROLE PASSWORD '<MIGRATOR_PASSWORD>';
CREATE ROLE nocturne_app      LOGIN NOSUPERUSER NOBYPASSRLS NOCREATEDB NOCREATEROLE PASSWORD '<APP_PASSWORD>';
CREATE ROLE nocturne_web      LOGIN NOSUPERUSER NOBYPASSRLS NOCREATEDB NOCREATEROLE PASSWORD '<WEB_PASSWORD>';
ALTER DATABASE nocturne OWNER TO nocturne_migrator;
ALTER SCHEMA public OWNER TO nocturne_migrator;
GRANT CONNECT ON DATABASE nocturne TO nocturne_app;
GRANT USAGE ON SCHEMA public TO nocturne_app;
GRANT CONNECT ON DATABASE nocturne TO nocturne_web;
GRANT USAGE, CREATE ON SCHEMA public TO nocturne_web;
ALTER DEFAULT PRIVILEGES FOR ROLE nocturne_migrator IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO nocturne_app;
ALTER DEFAULT PRIVILEGES FOR ROLE nocturne_migrator IN SCHEMA public
    GRANT USAGE, SELECT ON SEQUENCES TO nocturne_app;
SQL
```

The `<MIGRATOR_PASSWORD>`, `<APP_PASSWORD>`, and `<WEB_PASSWORD>` values correspond to `Passwords.1`, `Passwords.2`, and `Passwords.3` in the template — visible as `ConnectionStrings__*` and `NOCTURNE_POSTGRES_URI` environment variables on the running containers.

After running the SQL, restart the `<name>-api` and `<name>-web` containers.

### Post-deployment configuration

Two environment variables are left empty by the template because Cosmos has no hostname template variable — they must be set after deployment:

1. In the Cosmos UI, open each container's **Environment** settings.
2. On the **`<name>-api`** container, set `BASE_DOMAIN` to your hostname (e.g. `nocturne.example.com`).
3. On the **`<name>-web`** container, set both `BASE_DOMAIN` and `ORIGIN` (e.g. `https://nocturne.example.com`).
4. Restart both containers.

> `ORIGIN` is required by SvelteKit for CSRF protection. The app will reject form submissions until it is set correctly.

Once running, log in and configure data connectors and chat bot integrations through **Settings → Administration**.

## Development

### Running Tests

```bash
# Run all tests
dotnet test

# Run unit tests only
dotnet test --filter "Category!=Integration&Category!=Performance"

# Run with coverage
dotnet test --collect:"XPlat Code Coverage"
```

### Database Migrations

```bash
# Create a new migration
cd src/Infrastructure/Nocturne.Infrastructure.Data
dotnet ef migrations add YourMigrationName

# Apply migrations
dotnet ef database update
```

## API Documentation

API documentation is available via [Scalar](https://scalar.com/) at `https://localhost:1612/scalar` when running locally.

Nocturne aims to match Nightscout's API 1:1, so any Nightscout API endpoint should be usable. Nocturne-only endpoints are scoped to v4.

## Other stuff

### License

Nocturne is licensed under the [GNU Affero General Public License v3.0 (AGPL-3.0)](LICENSE). Commercial licensing is available for organizations that need to use Nocturne without AGPL obligations — contact the maintainers for details.

### Disclaimer

Nocturne is a community project and is not affiliated with or endorsed by the Nightscout Project, Abbott, Dexcom, Medtronic, Glooko, or MyFitnessPal.

**Important:** This software is provided as-is for personal use. Always verify glucose readings with approved medical devices. Never make treatment decisions based solely on data from this application.

### Support us

Nocturne is a labor of love built by volunteers. If you find it useful, please consider supporting the project:

- ⭐ Star the repository on GitHub
- [Donate to the Nightscout Foundation](https://nightscoutfoundation.org/donate)
- Support the maintainers on GitHub Sponsors!

### Acknowledgments

- Built on the shoulders of the [Nightscout Project](https://github.com/nightscout/cgm-remote-monitor)
- Powered by [.NET 10](https://dotnet.microsoft.com/) and [Aspire](https://learn.microsoft.com/en-us/dotnet/aspire/)
