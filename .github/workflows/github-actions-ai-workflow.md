# GitHub Actions Workflow: `ai.yml`

This workflow automates build, test, and cache cleanup for a Node.js project using GitHub Actions. It is triggered every time you push to the repository.

---

## 📌 Trigger

```yaml
on: push
```

- This means the workflow runs every time you push code to any branch in this repository.

---

## 🧱 Job: `build-and-test`

```yaml
jobs:
  build-and-test:
    runs-on: ubuntu-latest
```

- Runs on a fresh Ubuntu virtual machine.

### 🪜 Steps:

#### 1. Checkout Code

```yaml
- name: Checkout code
  uses: actions/checkout@v3
```
- This pulls your source code from the GitHub repository into the runner.

#### 2. Setup Node.js

```yaml
- name: Setup Node.js
  uses: actions/setup-node@v3
  with:
    node-version: "20.19.0"
```
- Installs Node.js version 20.19.0.

#### 3. Cache `node_modules`

```yaml
- name: Cache node_modules
  id: node_modules-cache
  uses: actions/cache@v3
  with:
    path: node_modules
    key: ${{ runner.os }}-node-modules-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-modules-
```
- Reuses `node_modules` from previous runs to speed up the workflow.
- The cache key is based on the hash of your `package-lock.json`.

#### 4. Show Cache Status

```yaml
- name: Show cache status
  run: echo "Cache hit ${{ steps.node_modules-cache.outputs.cache-hit }}"
```
- Logs whether the cache was used.

#### 5. Fix esbuild Permission (Only if cache hit)

```yaml
- name: Fix esbuild permissions (before install)
  if: steps.node_modules-cache.outputs.cache-hit == 'true'
  run: |
    if [ -f node_modules/esbuild/bin/esbuild ]; then
      echo "Fixing permissions on esbuild binary"
      chmod +x node_modules/esbuild/bin/esbuild
    fi
```
- Fixes permission issue with esbuild if it was restored from cache.

#### 6. Install Dependencies

```yaml
- name: Install dependencies
  run: npm ci
```
- Runs a clean install of dependencies.

#### 7. Run Tests

```yaml
- name: Run tests
  run: npm run test
```
- Executes test cases.

#### 8. Build

```yaml
- name: Build
  run: npm run build
```
- Builds the project (e.g., for production or distribution).

#### 9. Upload node_modules (optional)

```yaml
- name: Save node_modules as artifact
  if: steps.node_modules-cache.outputs.cache-hit != 'true'
  uses: actions/upload-artifact@v4
  with:
    name: node_modules
    path: node_modules/
```
- Shares `node_modules` with the next job by uploading it as an artifact.

---

## 🔁 Job: `test-again`

```yaml
test-again:
  runs-on: ubuntu-latest
  needs: build-and-test
```
- Runs after `build-and-test`.

### Steps:

#### 1. Checkout and Setup Node.js

```yaml
- uses: actions/checkout@v3
- uses: actions/setup-node@v3
  with:
    node-version: "20.19.0"
```

#### 2. Download node_modules artifact

```yaml
- name: Download node_modules artifact
  uses: actions/download-artifact@v4
  with:
    name: node_modules
    path: node_modules/
```

#### 3. Fix esbuild Permission (after artifact restore)

```yaml
- name: Fix esbuild permissions (after artifact restore)
  run: |
    if [ -f node_modules/esbuild/bin/esbuild ]; then
      chmod +x node_modules/esbuild/bin/esbuild
    fi
```

#### 4. Install Vitest and Run Additional Tests

```yaml
- name: Install Vitest and run additional tests
  run: |
    npm install --save-dev vitest
    npm run test
```

---

## 🧹 Job: `clear-cache`

```yaml
clear-cache:
  runs-on: ubuntu-latest
  needs: [build-and-test, test-again]
  if: always()
  permissions:
    actions: write
```
- Runs after both previous jobs, even if one fails.
- Deletes all cache keys for the current branch.

### Steps:

#### 1. Setup GitHub CLI

```yaml
- uses: sersoft-gmbh/setup-gh-cli-action@v2
```

#### 2. List and Delete Cache IDs

```yaml
- name: Delete branch-specific caches
  env:
    GH_TOKEN: ${{ github.token }}
  run: |
    echo "Fetching cache IDs for branch: ${{ github.ref_name }}"
    CACHE_IDS=$(gh cache list --repo "${{ github.repository }}" --ref "refs/heads/${{ github.ref_name }}" --limit 100 --json id --jq '.[].id')
    ...
```
- Uses `gh cache` to list and delete caches.

---

## 🧪 Job: `report`

```yaml
report:
  runs-on: ubuntu-latest
  needs: [build-and-test, test-again, clear-cache]
  if: failure()
```
- Runs only if one of the previous jobs fails.

### Step:

```yaml
- name: Output debug info
  run: |
    echo " Workflow failed"
    echo "${{ toJSON(github) }}"
```
- Prints debugging information.

---

## ✅ Summary

This workflow is optimized for:

- Speed (via caching)
- Reliability (via permissions fixes)
- Cleanliness (via branch-specific cache cleanup)
- Observability (via failure reporting)