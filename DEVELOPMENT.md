# Development Guide

This guide will help you set up and run the x402scan project for local development.

## Prerequisites

Before you begin, ensure you have the following installed:

- **Node.js**: Version **18.18.0 or higher** (Next.js requires `^18.18.0 || ^19.8.0 || >= 20.0.0`)
  - Check your version: `node --version`
  - Recommended: Node.js 20.x LTS or higher
  - Install/update using [nvm](https://github.com/nvm-sh/nvm) (recommended):
    ```bash
    nvm install 20
    nvm use 20
    ```
  - Or download from [nodejs.org](https://nodejs.org/)
- **pnpm**: Version 10.11.0 (specified in `package.json`)
  - Install with: `npm install -g pnpm@10.11.0` or `corepack enable && corepack prepare pnpm@10.11.0 --activate`
- **PostgreSQL**: For the main database (scan-db)
- **PostgreSQL**: For the transfers database (transfers-db)
- **Redis** (optional): For caching
- **ClickHouse** (optional): For analytics database

## Project Structure

This is a pnpm monorepo managed with Turbo. The main workspaces are:

- **`apps/scan/`** - Next.js web application (the main frontend)
- **`apps/proxy/`** - Hono-based proxy server
- **`apps/rpcs/solana/`** - Cloudflare Workers RPC service for Solana
- **`sync/transfers/`** - Background sync service for transfers (Trigger.dev)
- **`sync/analytics/`** - Analytics cron jobs (Trigger.dev)
- **`sync/alerts/`** - Balance alerts service (Trigger.dev)
- **`packages/internal/databases/scan/`** - Prisma schema and client for main database
- **`packages/internal/databases/transfers/`** - Prisma schema and client for transfers database
- **`packages/internal/databases/analytics/`** - ClickHouse client for analytics
- **`packages/external/facilitators/`** - Shared facilitator configuration

## Initial Setup

### 1. Clone the Repository

```bash
git clone <repository-url>
cd x402scan
```

### 2. Install Dependencies

```bash
pnpm install
```

This will install all dependencies for all workspaces in the monorepo.

### 3. Set Up Databases

**Important**: Prisma reads `.env` files from disk (not shell environment variables). Prisma searches for `.env` files starting from the schema directory and walking up to the root. For this monorepo, you should create a `.env` file at the **root** of the monorepo (see Environment Variables section below).

#### Main Database (scan-db)

The main application uses PostgreSQL with Prisma. You need to:

1. Create a PostgreSQL database
2. Set up connection strings in your `.env` file at the root of the monorepo (see Environment Variables section)
3. Run migrations from the root:

```bash
# Recommended: Run from root (Prisma will find .env at root)
pnpm --filter @x402scan/scan-db db:migrate
```

Or if you need to run from the database directory, ensure `.env` is at the root:

```bash
cd packages/internal/databases/scan
pnpm db:migrate
```

#### Transfers Database (transfers-db)

The transfers service uses a **separate** PostgreSQL database optimized for time-series blockchain event data. This is separate from the scan database for performance, scaling, and isolation reasons.

**Setup Options:**

**Option 1: Same Neon Project, Different Database (Recommended for Development)**
- In your Neon project, create a second database (e.g., `transfers_db`)
- Use the same connection endpoint but with a different database name:

```env
# Scan database
POSTGRES_PRISMA_URL="postgresql://user:pass@ep-xxx-pooler.region.aws.neon.tech/neondb?sslmode=require"
POSTGRES_URL_NON_POOLING="postgresql://user:pass@ep-xxx.region.aws.neon.tech/neondb?sslmode=require"

# Transfers database (same project, different database)
TRANSFERS_DB_URL="postgresql://user:pass@ep-xxx-pooler.region.aws.neon.tech/transfers_db?sslmode=require"
```

**Option 2: Separate Neon Project (Recommended for Production)**
- Create a completely separate Neon project for transfers
- Use a different connection endpoint:

```env
# Scan database
POSTGRES_PRISMA_URL="postgresql://user:pass@ep-xxx-pooler.region.aws.neon.tech/neondb?sslmode=require"
POSTGRES_URL_NON_POOLING="postgresql://user:pass@ep-xxx.region.aws.neon.tech/neondb?sslmode=require"

# Transfers database (separate project)
TRANSFERS_DB_URL="postgresql://user:pass@ep-yyy-pooler.region.aws.neon.tech/transfers_db?sslmode=require"
```

**Why Separate Databases?**
- **Different schemas**: Scan DB stores application data (Users, Resources, Sessions), while Transfers DB stores blockchain events (TransferEvent)
- **Performance isolation**: Prevents one workload from affecting the other
- **Independent scaling**: Transfers DB supports read replicas for heavy query loads
- **Different optimization**: Transfers DB is optimized for time-series queries

After setting up your connection strings, run migrations:

```bash
# Recommended: Run from root
pnpm --filter @x402scan/transfers-db db:migrate
```

Or from the database directory:

```bash
cd packages/internal/databases/transfers
pnpm db:migrate
```

#### Analytics Database (analytics-db) - Optional

If you want to use analytics features, set up ClickHouse:

1. Install and run ClickHouse
2. Configure connection details in environment variables

### 4. Generate Prisma Clients

Prisma clients are automatically generated during `pnpm install` via postinstall scripts, but you can regenerate them manually:

```bash
# For scan-db
pnpm --filter @x402scan/scan-db db:generate

# For transfers-db
pnpm --filter @x402scan/transfers-db db:generate
```

### 5. Environment Variables

**Best Practice**: Use a **single `.env` file at the root** of the monorepo. This is the proper way to manage environment variables in a monorepo.

**Setup**:

1. Create a `.env` file at the **root** of the monorepo with all your environment variables
2. Create a symlink for Next.js (which expects `.env` in the app directory):

```bash
# From the root directory
# Create your .env file at root, then:
ln -s ../../.env apps/scan/.env
```

**Why this works**:
- **Prisma** searches for `.env` files starting from the schema directory (`packages/internal/databases/scan/`) and walks up to the root, so it will find the root `.env`
- **Next.js** looks for `.env` in the app directory, so the symlink satisfies that requirement
- **Single source of truth**: Only one `.env` file to maintain and update

**File Structure**:
```
x402scan/
├── .env                    ← Single source of truth (actual file)
└── apps/
    └── scan/
        └── .env            ← Symlink to ../../.env
```

**Updating Environment Variables**:
Simply edit the root `.env` file - both Prisma and Next.js will automatically use the updated values (no copying needed!).

Create a `.env` file at the **root** of the monorepo with the following variables:

#### Required Variables

```env
# Database URLs
POSTGRES_PRISMA_URL="postgresql://user:password@localhost:5432/scan_db?pgbouncer=true"
POSTGRES_URL_NON_POOLING="postgresql://user:password@localhost:5432/scan_db"
TRANSFERS_DB_URL="postgresql://user:password@localhost:5432/transfers_db"

# Coinbase Developer Platform (CDP) Configuration
CDP_API_KEY_NAME="your-cdp-api-key-name"
CDP_API_KEY_ID="your-cdp-api-key-id"
CDP_API_KEY_SECRET="your-cdp-api-key-secret"
CDP_WALLET_SECRET="your-cdp-wallet-secret"
FREE_TIER_WALLET_NAME="your-wallet-name"

# Application URLs
NEXT_PUBLIC_APP_URL="http://localhost:3000"
NEXT_PUBLIC_PROXY_URL="http://localhost:3001"
NEXT_PUBLIC_NODE_ENV="development"

# NextAuth Configuration
AUTH_SECRET="generate-a-random-secret-here"  # Generate with: openssl rand -base64 32
```

#### Optional Variables

```env
# Redis (for caching)
REDIS_URL="redis://localhost:6379"

# Analytics (ClickHouse)
ANALYTICS_CLICKHOUSE_URL="http://localhost:8123"
ANALYTICS_CLICKHOUSE_USER="default"
ANALYTICS_CLICKHOUSE_PASSWORD=""
ANALYTICS_CLICKHOUSE_DATABASE="analytics"

# CDP Public Configuration
NEXT_PUBLIC_CDP_PROJECT_ID="your-project-id"
NEXT_PUBLIC_CDP_APP_ID="your-app-id"

# PostHog Analytics
NEXT_PUBLIC_POSTHOG_KEY="your-posthog-key"
NEXT_PUBLIC_POSTHOG_HOST="https://us.i.posthog.com"

# RPC URLs
NEXT_PUBLIC_SOLANA_RPC_URL="https://api.mainnet-beta.solana.com"
NEXT_PUBLIC_BASE_RPC_URL="https://mainnet.base.org"

# Feature Flags
NEXT_PUBLIC_ENABLE_COMPOSER="true"

# API Keys (optional)
GITHUB_TOKEN="your-github-token"
FREEPIK_API_KEY="your-freepik-key"
BLOB_READ_WRITE_TOKEN="your-vercel-blob-token"
JINA_API_KEY="your-jina-key"
RESOURCE_SEARCH_API_KEY="your-resource-search-key"

# Echo Proxy
ECHO_APP_ID="7fed205e-3aa5-44af-83a3-f7ae5e49dba4"
ECHO_PROXY_URL="https://your-echo-proxy-url.com"

# Transfers DB Replicas (optional, for read scaling)
TRANSFERS_DB_URL_REPLICA_1="postgresql://..."
TRANSFERS_DB_URL_REPLICA_2="postgresql://..."
TRANSFERS_DB_URL_REPLICA_3="postgresql://..."
TRANSFERS_DB_URL_REPLICA_4="postgresql://..."
TRANSFERS_DB_URL_REPLICA_5="postgresql://..."

# Development Only
HIDE_TRPC_LOGS="false"
```

#### Generate AUTH_SECRET

You can generate a secure `AUTH_SECRET` using:

```bash
openssl rand -base64 32
```

## Running the Development Servers

### Main Application (Next.js)

From the root directory:

```bash
pnpm dev:scan
```

Or from the `apps/scan/` directory:

```bash
cd apps/scan
pnpm dev
```

The application will be available at `http://localhost:3000`.

### Proxy Server

From the root directory:

```bash
cd apps/proxy
pnpm dev
```

The proxy server will run on port 3001 (or as configured).

### Sync Services (Trigger.dev)

The sync services use Trigger.dev for background jobs. To run them locally:

#### Transfers Sync

```bash
cd sync/transfers
pnpm trigger:dev
```

#### Analytics Sync

```bash
cd sync/analytics
pnpm trigger:dev
```

#### Alerts Sync

```bash
cd sync/alerts
pnpm trigger:dev
```

**Note**: You'll need a Trigger.dev account and `TRIGGER_ACCESS_TOKEN` environment variable to run sync services.

### Solana RPC Service (Cloudflare Workers)

To run the Solana RPC service locally:

```bash
cd apps/rpcs/solana
pnpm dev
```

This uses Wrangler for local development. Make sure you have Cloudflare Workers configured.

## Database Management

### Prisma Studio

You can use Prisma Studio to view and edit your database:

```bash
# For scan-db
cd packages/internal/databases/scan
pnpm studio

# For transfers-db
cd packages/internal/databases/transfers
pnpm studio
```

### Running Migrations

```bash
# Scan database
pnpm --filter @x402scan/scan-db db:migrate

# Transfers database
pnpm --filter @x402scan/transfers-db db:migrate
```

### Creating New Migrations

```bash
# Scan database
cd packages/internal/databases/scan
pnpm db:migrate --name your_migration_name

# Transfers database
cd packages/internal/databases/transfers
pnpm db:migrate --name your_migration_name
```

## Available Scripts

### Root Level Scripts

- `pnpm dev:scan` - Run the Next.js development server
- `pnpm build:scan` - Build the Next.js application
- `pnpm build:proxy` - Build the proxy server
- `pnpm build` - Build all workspaces
- `pnpm lint` - Lint all workspaces
- `pnpm format` - Format all code with Prettier
- `pnpm check:format` - Check code formatting
- `pnpm check:types` - Type check all workspaces
- `pnpm check` - Run all checks (format, types, lint, knip)

### Scan App Scripts

From `apps/scan/`:

- `pnpm dev` - Start development server with Turbopack
- `pnpm build` - Build for production
- `pnpm start` - Start production server
- `pnpm lint` - Run ESLint
- `pnpm typegen` - Generate Next.js types
- `pnpm test` - Run tests with Vitest
- `pnpm sync:ecosystem` - Sync ecosystem data

## Development Workflow

### Making Changes

1. Create a new branch for your changes
2. Make your changes in the appropriate workspace
3. Run type checking: `pnpm check:types`
4. Run linting: `pnpm lint`
5. Format code: `pnpm format`
6. Test your changes locally
7. Commit and push

### Hot Reloading

The Next.js app uses Turbopack for fast hot reloading. Changes to:
- React components will hot reload automatically
- Server components and API routes require a page refresh
- Database schema changes require regenerating Prisma client

### Type Safety

The project uses TypeScript with strict type checking. Make sure to:

1. Run `pnpm check:types` before committing
2. Regenerate Prisma clients after schema changes
3. Run `pnpm typegen` in the scan app after Next.js changes

## Troubleshooting

### Database Connection Issues

- Verify your PostgreSQL server is running
- Check connection strings in `.env`
- Ensure databases exist and migrations have been run
- Check firewall settings if using remote databases

### Database Connection Strings Not Updating

If you've updated your database connection strings in `apps/scan/.env` but Prisma is still using the old connection string:

**The Issue**: Prisma reads from `packages/internal/databases/scan/.env`, not from `apps/scan/.env`. You must copy the updated `.env` file to the database package directories.

**Solution**: After updating `apps/scan/.env`, always copy it to the database package directories:

```bash
# From the root directory
cp apps/scan/.env packages/internal/databases/scan/.env
cp apps/scan/.env packages/internal/databases/transfers/.env
```

**For Neon Databases**: Make sure you have both connection strings correctly configured:
- `POSTGRES_PRISMA_URL` should use the **pooler** endpoint (has `-pooler` in hostname)
- `POSTGRES_URL_NON_POOLING` should use the **direct** endpoint (no `-pooler`)

Example:
```env
POSTGRES_PRISMA_URL="postgresql://user:pass@ep-xxx-pooler.region.aws.neon.tech/db?sslmode=require"
POSTGRES_URL_NON_POOLING="postgresql://user:pass@ep-xxx.region.aws.neon.tech/db?sslmode=require"
```

### Prisma Can't Find Environment Variables

If you get errors like `Environment variable not found: POSTGRES_URL_NON_POOLING` when running migrations:

**The Issue**: Prisma reads `.env` files from disk (not shell environment variables). Sourcing a `.env` file with `source` won't work. Prisma searches for `.env` files starting from the schema directory and walking up to the root.

**Solution**: This project uses `dotenv-cli` to explicitly load the root `.env` file for Prisma commands. The database package scripts are already configured to use it.

According to the [Prisma GitHub discussion](https://github.com/prisma/prisma/discussions/15502), Prisma reads from the actual environment and has a helper that loads `.env` files, but in monorepos it may not always find the root `.env` file automatically.

**The Fix**: The project uses `dotenv-cli` in the database package scripts to explicitly load the root `.env`:

```json
"db:migrate": "dotenv -e ../../../../.env -- prisma migrate dev --skip-generate"
```

This ensures Prisma can always find the environment variables from the root `.env` file, regardless of where the command is run from.

**Running Migrations**:
```bash
# From root (recommended)
pnpm --filter @x402scan/scan-db db:migrate
pnpm --filter @x402scan/transfers-db db:migrate

# Or from the database directory
cd packages/internal/databases/scan
pnpm db:migrate
```

**Note**: You should have a single `.env` at root, not copies in multiple directories. See the Environment Variables section above for the proper setup.

**Verify**: Prisma searches in this order (from `packages/internal/databases/scan/`):
1. `packages/internal/databases/scan/.env`
2. `packages/internal/databases/.env`
3. `packages/internal/.env`
4. `packages/.env`
5. `.env` (root) ← **Recommended location**

### Prisma Client Not Found

If you see errors about Prisma client not being found:

```bash
pnpm --filter @x402scan/scan-db db:generate
pnpm --filter @x402scan/transfers-db db:generate
```

### Port Already in Use

If port 3000 is already in use, you can change it:

```bash
PORT=3001 pnpm dev:scan
```

Or update `NEXT_PUBLIC_APP_URL` in your `.env` file.

### Environment Variable Errors

The project uses `@t3-oss/env-nextjs` for environment variable validation. Make sure:

1. All required variables are set
2. URLs are valid (start with `http://` or `https://`)
3. No trailing slashes in URLs (unless required)

### Turbo Cache Issues

If you encounter caching issues with Turbo:

```bash
# Clear Turbo cache
rm -rf .turbo

# Or disable cache for a specific run
pnpm dev:scan --force
```

### Node.js Version Mismatch

If you get an error like:
```
You are using Node.js 18.14.2. For Next.js, Node.js version "^18.18.0 || ^19.8.0 || >= 20.0.0" is required.
```

**The Issue**: Your system may have multiple Node.js versions, and pnpm/turbo might be using a different version than your default.

**Solution 1**: Check which Node version pnpm is using:
```bash
pnpm exec node --version
```

**Solution 2**: Use nvm to ensure the correct Node version:
```bash
# If using nvm
nvm use 20
# Or install Node 20 if you don't have it
nvm install 20
nvm use 20
```

**Solution 3**: Set Node version in your shell profile (`.zshrc` or `.bashrc`):
```bash
# Add to ~/.zshrc or ~/.bashrc
export NODE_VERSION=20
```

**Solution 4**: Create a `.nvmrc` file at the root of the project:
```bash
echo "20" > .nvmrc
# Then run
nvm use
```

**Solution 5**: If using pnpm's Node version management:
```bash
# Check pnpm's Node version
pnpm env use --global 20
```

### Missing Dependencies

If you see missing dependency errors:

```bash
# Reinstall all dependencies
rm -rf node_modules
pnpm install
```

## Testing

Run tests with:

```bash
cd apps/scan
pnpm test
```

For watch mode:

```bash
pnpm test --watch
```

## Code Quality

### Linting

```bash
pnpm lint
```

### Formatting

```bash
# Format all code
pnpm format

# Check formatting
pnpm check:format
```

### Type Checking

```bash
pnpm check:types
```

### Dead Code Detection

The project uses Knip to detect unused code:

```bash
pnpm knip
```

## Additional Resources

- [Next.js Documentation](https://nextjs.org/docs)
- [Prisma Documentation](https://www.prisma.io/docs)
- [Trigger.dev Documentation](https://trigger.dev/docs)
- [Turbo Documentation](https://turbo.build/repo/docs)
- [pnpm Documentation](https://pnpm.io/)

## Getting Help

If you encounter issues:

1. Check this guide first
2. Review the existing README.md
3. Check GitHub issues
4. Reach out on Discord: [x402scan Discord](https://discord.gg/JuKt7tPnNc)

## Contributing

See the main README.md for contribution guidelines. Key points:

- Add resources via the web interface at `/resources/register`
- Add facilitators by editing `packages/external/facilitators/src/config.ts`
- Run `pnpm check:facilitators` to validate facilitator changes

