# CI/CD Pipeline Optimization Strategies

## 1. Dependency Caching

**npm/yarn cache:**
```yaml
- name: Cache dependencies
  uses: actions/cache@v3
  with:
    path: node_modules
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-npm-
```

**Docker layer caching:**
- Place frequently changing layers at the end
- Use BuildKit for better caching: `DOCKER_BUILDKIT=1 docker build`

## 2. Parallelism

**Run jobs concurrently:**
```yaml
jobs:
  test-unit:
    runs-on: ubuntu-latest
    steps:
      - run: npm run test:unit

  test-integration:
    runs-on: ubuntu-latest
    steps:
      - run: npm run test:integration

  lint:
    runs-on: ubuntu-latest
    steps:
      - run: npm run lint
```

**Matrix strategy for multiple environments:**
```yaml
jobs:
  test:
    strategy:
      matrix:
        node-version: [18, 20]
        os: [ubuntu-latest, windows-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node-version }}
```

## 3. Incremental Builds

**Only run tests on changed files:**
```bash
git diff --name-only HEAD~1 | grep '\.ts$' | xargs npm run test --
```

**Skip expensive steps conditionally:**
```yaml
- name: Run integration tests
  if: github.event_name == 'pull_request'
  run: npm run test:integration
```

## 4. Fail Fast

**Reorder checks by speed (fast first):**
1. Linting (seconds)
2. Unit tests (minutes)
3. Build (minutes)
4. Integration tests (slow)

**Exit early on lint failure:**
```yaml
- run: npm run lint
- run: npm run test:unit
- run: npm run build
- run: npm run test:integration
```

## 5. Resource Optimization

- Use appropriate runner size (smaller = faster startup)
- Minimize artifact storage
- Clean up workspace before/after

**Cleanup step:**
```yaml
- name: Clean workspace
  run: |
    rm -rf node_modules
    rm -rf dist
    df -h
```

## Performance Targets

- Small projects: 5-10 minutes
- Medium projects: 10-20 minutes
- Large projects: 20-30 minutes (with parallelism)
