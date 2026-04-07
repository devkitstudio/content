## Complete turbo.json

```json
{
  "$schema": "https://turborepo.org/schema.json",
  "globalDependencies": [
    "tsconfig.json",
    ".env",
    ".eslintrc.json"
  ],
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", "build/**", ".next/**"],
      "cache": true,
      "env": ["NODE_ENV", "PUBLIC_URL"]
    },
    "test": {
      "dependsOn": ["build"],
      "outputs": ["coverage/**"],
      "cache": true,
      "inputs": ["src/**", "test/**", "jest.config.js"]
    },
    "lint": {
      "outputs": [".eslintcache"],
      "cache": true,
      "inputs": ["src/**", ".eslintrc.json"]
    },
    "type-check": {
      "cache": true,
      "inputs": ["src/**", "tsconfig.json"]
    },
    "dev": {
      "cache": false,
      "persistent": true,
      "outputs": ["**/.next/**"]
    },
    "docs#generate": {
      "outputs": ["dist/**"],
      "cache": true
    }
  }
}
```

## Root package.json

```json
{
  "name": "my-monorepo",
  "private": true,
  "workspaces": [
    "apps/*",
    "packages/*"
  ],
  "scripts": {
    "build": "turbo run build",
    "test": "turbo run test",
    "lint": "turbo run lint",
    "type-check": "turbo run type-check",
    "dev": "turbo run dev --parallel",
    "clean": "turbo clean && rm -rf node_modules"
  },
  "devDependencies": {
    "turbo": "^1.11.0"
  }
}
```

## Package Scripts

```json
{
  "name": "@monorepo/web",
  "private": true,
  "scripts": {
    "build": "next build",
    "dev": "next dev -p 3000",
    "lint": "eslint src --ext ts,tsx",
    "type-check": "tsc --noEmit",
    "test": "jest"
  },
  "dependencies": {
    "@monorepo/ui": "*",
    "@monorepo/utils": "*",
    "@monorepo/api-client": "*"
  }
}
```

## Selective Building

```bash
# Build only web app and dependencies
turbo run build --filter=@monorepo/web

# Build except web app
turbo run build --filter='!@monorepo/web'

# Build only changed packages
turbo run build --filter=[main]

# Build package and dependents
turbo run build --filter=@monorepo/utils --filter=...@monorepo/utils
```

## CI/CD Integration (GitHub Actions)

```yaml
name: CI

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: pnpm/action-setup@v2
        with:
          version: 8

      - uses: actions/setup-node@v4
        with:
          node-version: 18
          cache: pnpm

      - run: pnpm install

      # Use remote cache
      - run: pnpm turbo login --token ${{ secrets.TURBO_TOKEN }}

      - run: pnpm turbo run lint type-check test build --remote

      - run: pnpm turbo run build --filter=[main]
```

## Debugging Cache Issues

```bash
# See task execution details
turbo run build --verbose

# Skip cache (force rebuild)
turbo run build --no-cache

# Check what's cached
turbo run build --dry

# Clear local cache
turbo clean

# View cache folder size
du -sh node_modules/.turbo/cache

# Profile turbo
turbo run build --profile=profile.json
```

## Performance Tips

1. **Minimize outputs**
   ```json
   "outputs": ["dist/**"]      // Specific
   "outputs": ["**"]           // Avoid!
   ```

2. **Specify inputs**
   ```json
   "inputs": ["src/**", "package.json"]  // Rebuild on these changes
   ```

3. **Use .gitignore**
   ```
   dist/
   .next/
   coverage/
   ```

4. **Separate build/test tasks**
   ```json
   "test": { "dependsOn": ["build"] }  // Only rebuild if build changed
   ```
