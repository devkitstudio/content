## Problem Without Turborepo

```bash
# Change one shared component
# Builds entire monorepo (3 apps + 20 packages)
npm run build        # 45 seconds - always
# Every build same time, whether you changed 1 file or all
```

## Turborepo Solution: Smart Caching & Dependency Graph

Turborepo:
1. **Understands dependencies** between packages
2. **Only rebuilds affected** packages
3. **Caches build artifacts** locally and remotely
4. **Parallelizes tasks** across packages
5. **Skips unchanged** packages

```bash
# With Turborepo:
turbo run build       # 45s first time
turbo run build       # 2s cached (only changed deps rebuilt)
turbo run build --filter=@app/dashboard  # 3s (dashboard + deps)
```

## Monorepo Structure

```
monorepo/
├── apps/
│   ├── web/           # Next.js app
│   ├── admin/         # React admin dashboard
│   └── mobile/        # React Native
├── packages/
│   ├── ui/            # Shared UI components
│   ├── utils/         # Utilities
│   ├── api-client/    # API layer
│   └── types/         # TS types
├── turbo.json         # Turborepo config
└── package.json       # Workspace root
```

## Workspace Setup

```json
{
  "name": "monorepo",
  "version": "1.0.0",
  "private": true,
  "workspaces": [
    "apps/*",
    "packages/*"
  ],
  "devDependencies": {
    "turbo": "latest"
  }
}
```

## Task Pipeline

```yaml
# turbo.json
{
  "globalDependencies": [".env", "tsconfig.json"],
  "tasks": {
    "build": {
      "dependsOn": ["^build"],          # Build deps first
      "outputs": ["dist/**", ".next/**"],
      "cache": true,
      "env": ["NODE_ENV"]
    },
    "test": {
      "dependsOn": ["build"],           # Test requires build
      "outputs": ["coverage/**"],
      "cache": true,
      "inputs": ["src/**", "test/**"]
    },
    "lint": {
      "cache": true
    },
    "dev": {
      "cache": false,                   # Never cache dev
      "persistent": true
    }
  }
}
```

**dependsOn: "^build"** = "Build my dependencies first"

## Dependency Graph

```
web (Next.js app)
├── ui (shared components)
│   ├── utils (helpers)
│   ├── types (types)
│   └── tailwind config
├── api-client
│   └── types
└── utils

admin (React admin)
├── ui (shared)
│   └── utils
├── api-client
│   └── types
└── utils
```

**Run build:**
1. types builds (no deps)
2. utils builds (depends on types)
3. ui, api-client build in parallel (both depend on types/utils)
4. web, admin build in parallel (both depend on all above)

**Without smart caching:** 45s always
**With caching:** 2s if only types changed

## Caching Strategy

```json
{
  "build": {
    "outputs": ["dist/**", ".next/**"],  // Cache these files
    "cache": true,
    "inputs": ["src/**", "tsconfig.json"], // Hash these inputs
    "env": ["NODE_ENV", "DATABASE_URL"]    // Include env vars in hash
  }
}
```

**Cache key includes:**
- Source files content hash
- Package dependencies
- Specified environment variables
- turbo.json configuration

If nothing changed, Turborepo skips task and restores cached output.

## Remote Caching (Team)

```bash
# Cache builds in cloud (Vercel, AWS S3)
turbo login                 # Sign in
turbo run build --remote    # Upload cache
turbo run build             # Download from cache automatically
```

Benefits:
- CI runs 5-10x faster
- Developers share cache
- First CI run fast
- Local + remote cache

## Performance Gains Example

```
First build (cold):           45s
Second build (cached):        2s
Change one file:              12s (rebuild affected only)
Change types (many deps):     18s (cascading rebuilds)
CI build (remote cache):      3s (download cache + light build)

Before Turborepo:
Every build:                  45s (no caching)
```
