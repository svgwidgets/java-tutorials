# GitLab NPM Publishing with Semantic Release — Complete Guide

## Table of Contents

1. [Architecture & Best Practices](#1-architecture--best-practices)
2. [`.gitlab-ci.yml`](#2-gitlab-ciyml)
3. [`package.json`](#3-packagejson)
4. [`.npmrc` (CI + Consumer)](#4-npmrc)
5. [Semantic Release Config](#5-semantic-release-config)
6. [Token Strategy](#6-token-strategy)
7. [Troubleshooting](#7-troubleshooting)

---

## 1. Architecture & Best Practices

### Workflow Overview

```
developer pushes feat: commit
        │
        ▼
  feature branch CI ──► install ► lint ► test ► build  (no publish)
        │
        ▼
  MR merged to master
        │
        ▼
  master CI pipeline:
    install ► lint ► test ► build ► semantic-release
                                        │
                            ┌───────────┼───────────┐
                            ▼           ▼           ▼
                     bump version   git tag    npm publish
                     in package.json  v1.2.3   to GitLab
                            │           │       Package
                            ▼           ▼       Registry
                       GitLab Release
                       with changelog
```

### Branch Rules

| Rule | Setting |
|------|---------|
| Default branch | `master` |
| Protected branch | `master` — Maintainers can push, no force push |
| Protected tags | `v*` — Only Maintainers can create |
| Merge method | Merge commit or squash (squash recommended for clean history) |

> **Why protect `v*` tags?** semantic-release creates tags like `v1.2.3`. If anyone can create tags, a developer could accidentally tag and cause version conflicts.

### Commit Convention (Conventional Commits)

semantic-release reads commit messages to determine version bumps:

| Prefix | Version Bump | Example |
|--------|-------------|---------|
| `fix:` | PATCH (1.0.0 → 1.0.1) | `fix: correct tank fill animation` |
| `feat:` | MINOR (1.0.0 → 1.1.0) | `feat: add butterfly valve component` |
| `feat!:` or `BREAKING CHANGE:` | MAJOR (1.0.0 → 2.0.0) | `feat!: redesign I/O abstraction API` |
| `chore:`, `docs:`, `ci:` | No release | `docs: update README` |

**Enforce with commitlint** (optional but recommended):

```bash
npm install -D @commitlint/{cli,config-conventional}
```

```js
// commitlint.config.js
module.exports = { extends: ['@commitlint/config-conventional'] };
```

---

## 2. `.gitlab-ci.yml`

```yaml
# ============================================================
# .gitlab-ci.yml — P&ID Component Library
# Publishes scoped npm package to GitLab Package Registry
# via semantic-release on master branch only.
# ============================================================

stages:
  - install
  - quality
  - build
  - release

# ── Global defaults ──────────────────────────────────────────
default:
  image: node:20-alpine
  cache:
    key:
      files:
        - package-lock.json
    paths:
      - node_modules/
    policy: pull

# ── Variables ────────────────────────────────────────────────
variables:
  # npm config for GitLab registry auth (used by semantic-release npm plugin)
  NPM_CONFIG_REGISTRY: "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/packages/npm/"

# ============================================================
# Stage: install
# ============================================================
install:
  stage: install
  script:
    - npm ci --prefer-offline
  cache:
    key:
      files:
        - package-lock.json
    paths:
      - node_modules/
    policy: pull-push   # populate cache

# ============================================================
# Stage: quality
# ============================================================
lint:
  stage: quality
  needs: [install]
  script:
    - npm run lint
  allow_failure: false

test:
  stage: quality
  needs: [install]
  script:
    - npm run test -- --run
  coverage: '/All files[^|]*\|[^|]*\s+([\d\.]+)/'
  artifacts:
    when: always
    reports:
      junit: junit.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml
    expire_in: 7 days
  allow_failure: false

# ============================================================
# Stage: build
# ============================================================
build:
  stage: build
  needs: [install]
  script:
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 day

# ============================================================
# Stage: release  (master only)
# ============================================================
release:
  stage: release
  needs: [lint, test, build]
  rules:
    - if: $CI_COMMIT_BRANCH == "master"
      when: on_success
    - when: never
  before_script:
    # semantic-release needs git for tagging
    - apk add --no-cache git
    # Configure npm auth for publishing
    - |
      echo "@my-group:registry=${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/packages/npm/"  > .npmrc
      echo "//${CI_SERVER_HOST}/api/v4/projects/${CI_PROJECT_ID}/packages/npm/:_authToken=${CI_JOB_TOKEN}" >> .npmrc
    # Configure git so semantic-release can push tags
    - git config user.email "ci@gitlab.com"
    - git config user.name "GitLab CI"
    # Set remote with token auth for tag pushing
    - git remote set-url origin "https://gitlab-ci-token:${GITLAB_TOKEN}@${CI_SERVER_HOST}/${CI_PROJECT_PATH}.git"
  script:
    - npx semantic-release
  artifacts:
    paths:
      - dist/
    expire_in: 30 days
```

### Key Points

- **`rules:`** ensures publish only runs on `master`. No `only/except` (deprecated).
- **`needs:`** ensures release waits for lint + test + build to all pass.
- **`CI_JOB_TOKEN`** is used for npm publish (auto-injected, no setup needed).
- **`GITLAB_TOKEN`** (a Project Access Token) is used for git tag push + GitLab release creation. See [Token Strategy](#6-token-strategy).
- **`apk add git`** — needed in alpine images for semantic-release to read git history.
- **Duplicate publish protection** — semantic-release checks if the version tag already exists. If it does, it skips. The npm registry also rejects duplicate versions with HTTP 409.

---

## 3. `package.json`

```jsonc
{
  "name": "@my-group/pid-components",
  "version": "0.0.0-semantically-released",
  "description": "Vue 3 P&ID Component Library for SCADA systems",
  "license": "MIT",
  "type": "module",

  // ── Entry points ───────────────────────────────────────
  "main": "./dist/pid-components.umd.cjs",
  "module": "./dist/pid-components.es.js",
  "types": "./dist/types/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/pid-components.es.js",
      "require": "./dist/pid-components.umd.cjs",
      "types": "./dist/types/index.d.ts"
    },
    "./style.css": "./dist/style.css"
  },
  "files": [
    "dist"
  ],

  // ── CRITICAL: publishConfig ────────────────────────────
  // This tells `npm publish` WHERE to publish.
  // Must match your GitLab project's npm endpoint.
  "publishConfig": {
    "@my-group:registry": "https://gitlab.com/api/v4/projects/YOUR_PROJECT_ID/packages/npm/"
  },

  // ── Repository (used by semantic-release gitlab plugin) ─
  "repository": {
    "type": "git",
    "url": "https://gitlab.com/my-group/pid-components.git"
  },

  // ── Scripts ────────────────────────────────────────────
  "scripts": {
    "dev": "vite",
    "build": "vite build && vue-tsc --emitDeclarationOnly",
    "lint": "eslint src/",
    "test": "vitest",
    "test:coverage": "vitest run --coverage"
  },

  // ── Dependencies ───────────────────────────────────────
  "peerDependencies": {
    "vue": "^3.3.0"
  },
  "devDependencies": {
    "@vitejs/plugin-vue": "^5.0.0",
    "semantic-release": "^24.0.0",
    "@semantic-release/changelog": "^6.0.3",
    "@semantic-release/git": "^10.0.1",
    "@semantic-release/gitlab": "^13.0.0",
    "@semantic-release/npm": "^12.0.0",
    "typescript": "^5.4.0",
    "vite": "^5.0.0",
    "vitest": "^2.0.0",
    "vue": "^3.4.0",
    "vue-tsc": "^2.0.0"
  }
}
```

### Important Notes

| Field | Why |
|-------|-----|
| `"version": "0.0.0-semantically-released"` | semantic-release overwrites this. Never manually change it. |
| `"name": "@my-group/..."` | Must be scoped to your GitLab top-level group/namespace. |
| `"publishConfig"` | Tells npm where to PUT the package. The project ID is numeric (find it in Settings → General). |
| `"files": ["dist"]` | Only ship built output, not source. |
| `"repository.url"` | The `@semantic-release/gitlab` plugin uses this to find your project. |

### Vite Library Build Config

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import vue from '@vitejs/plugin-vue';
import { resolve } from 'path';

export default defineConfig({
  plugins: [vue()],
  build: {
    lib: {
      entry: resolve(__dirname, 'src/lib/index.ts'),
      name: 'PidComponents',
      fileName: 'pid-components',
      formats: ['es', 'umd'],
    },
    rollupOptions: {
      external: ['vue'],
      output: {
        globals: { vue: 'Vue' },
      },
    },
  },
  resolve: {
    alias: {
      '@': resolve(__dirname, 'src'),
    },
  },
});
```

---

## 4. `.npmrc`

### 4a. `.npmrc` for CI Publishing (generated in pipeline)

This is created dynamically in `.gitlab-ci.yml`'s `before_script`. You do **not** commit this file. But for reference, the pipeline generates:

```ini
# Scoped registry for your group
@my-group:registry=https://gitlab.com/api/v4/projects/YOUR_PROJECT_ID/packages/npm/

# Auth token for that registry endpoint
//gitlab.com/api/v4/projects/YOUR_PROJECT_ID/packages/npm/:_authToken=${CI_JOB_TOKEN}
```

### 4b. `.npmrc` for Consumers (other teams installing the package)

Consumers need to create a `.npmrc` in their project root (or `~/.npmrc` globally):

```ini
# Option A: Project-level registry (recommended for most setups)
# Uses the GROUP-level npm endpoint so all packages in the group are accessible.
@my-group:registry=https://gitlab.com/api/v4/groups/YOUR_GROUP_ID/-/packages/npm/

# Auth — use a Deploy Token or Personal Access Token
//gitlab.com/api/v4/groups/YOUR_GROUP_ID/-/packages/npm/:_authToken=YOUR_TOKEN_HERE
```

**OR** using project-level endpoint:

```ini
# Option B: Project-level endpoint (if consumer only needs this one package)
@my-group:registry=https://gitlab.com/api/v4/projects/YOUR_PROJECT_ID/packages/npm/

//gitlab.com/api/v4/projects/YOUR_PROJECT_ID/packages/npm/:_authToken=YOUR_TOKEN_HERE
```

Then consumers install normally:

```bash
npm install @my-group/pid-components
# or
yarn add @my-group/pid-components
# or
pnpm add @my-group/pid-components
```

### Consumer Token Options

| Token Type | Scope | Best For |
|-----------|-------|----------|
| **Deploy Token** (recommended) | `read_package_registry` | CI/CD of consuming projects |
| **Personal Access Token** | `read_api` | Individual developer machines |
| **CI_JOB_TOKEN** | Automatic | Same-instance CI pipelines (limited cross-project) |

> **Pro tip:** For cross-project CI usage, create a **Group Deploy Token** with `read_package_registry` scope. All projects in the group can use it.

---

## 5. Semantic Release Config

Create `.releaserc.json` in your project root:

```json
{
  "branches": ["master"],
  "tagFormat": "v${version}",
  "plugins": [
    ["@semantic-release/commit-analyzer", {
      "preset": "angular",
      "releaseRules": [
        { "type": "feat", "release": "minor" },
        { "type": "fix", "release": "patch" },
        { "type": "perf", "release": "patch" },
        { "type": "revert", "release": "patch" },
        { "type": "refactor", "release": "patch" },
        { "breaking": true, "release": "major" }
      ]
    }],

    "@semantic-release/release-notes-generator",

    ["@semantic-release/changelog", {
      "changelogFile": "CHANGELOG.md"
    }],

    ["@semantic-release/npm", {
      "npmPublish": true
    }],

    ["@semantic-release/git", {
      "assets": ["CHANGELOG.md", "package.json"],
      "message": "chore(release): ${nextRelease.version} [skip ci]\n\n${nextRelease.notes}"
    }],

    ["@semantic-release/gitlab", {
      "gitlabUrl": "https://gitlab.com",
      "assets": []
    }]
  ]
}
```

### Plugin Execution Order (matters!)

| Order | Plugin | What It Does |
|-------|--------|-------------|
| 1 | `commit-analyzer` | Reads commits since last tag → determines next version |
| 2 | `release-notes-generator` | Generates markdown release notes from commits |
| 3 | `changelog` | Writes/updates `CHANGELOG.md` |
| 4 | `npm` | Runs `npm publish` (publishes to the registry set in `.npmrc`) |
| 5 | `git` | Commits updated `CHANGELOG.md` + `package.json`, pushes |
| 6 | `gitlab` | Creates a GitLab Release (visible in Deploy → Releases) with notes |

### Notes

- **`[skip ci]`** in the git commit message prevents the release commit from triggering another pipeline (avoids infinite loop).
- **`tagFormat: "v${version}"`** — creates tags like `v1.2.3`. Must match your protected tag pattern `v*`.
- **`@semantic-release/gitlab`** — set `gitlabUrl` to your instance URL. For self-managed, change to `https://your-gitlab.example.com`.
- **`npmPublish: true`** — the npm plugin handles both `npm version` and `npm publish`.

---

## 6. Token Strategy

### Tokens Needed

You need **two** tokens for the release job:

| Purpose | Token | How |
|---------|-------|-----|
| **npm publish** to Package Registry | `CI_JOB_TOKEN` | Auto-injected by GitLab. No setup needed. Has `write_package_registry` on the current project. |
| **Git push** (tags + release commit) + **GitLab Release creation** | `GITLAB_TOKEN` (Project Access Token) | Must be created manually. See below. |

### Step-by-Step: Create the `GITLAB_TOKEN`

1. **Go to:** Your GitLab project → Settings → Access Tokens
2. **Create token:**
   - **Name:** `semantic-release-bot`
   - **Role:** `Maintainer`
   - **Scopes:** ✅ `api`, ✅ `write_repository`
   - **Expiration:** Set a reasonable date (e.g., 1 year), and create a calendar reminder to rotate it
3. **Copy the token value**
4. **Go to:** Settings → CI/CD → Variables
5. **Add variable:**
   - **Key:** `GITLAB_TOKEN`
   - **Value:** (paste the token)
   - **Type:** Variable
   - **Flags:** ✅ Masked, ✅ Protected
   - **Environment scope:** `*` (or just `production` if you have environments)

> **Why `GITLAB_TOKEN` specifically?** The `@semantic-release/gitlab` plugin looks for the `GITLAB_TOKEN` (or `GL_TOKEN`) environment variable by convention. No extra config needed — it just works.

### Why Not Just `CI_JOB_TOKEN` for Everything?

`CI_JOB_TOKEN` limitations:
- **Cannot push git tags** to the repo (no `write_repository` permission)
- **Cannot create GitLab Releases** via the API
- **Cannot push commits** (needed for `@semantic-release/git` to commit CHANGELOG.md)

So you need `CI_JOB_TOKEN` for npm publish + a Project Access Token for git/gitlab operations.

### Self-Managed GitLab Differences

| Aspect | GitLab SaaS | Self-Managed |
|--------|------------|-------------|
| Registry URL | `https://gitlab.com/api/v4/...` | `https://your-gitlab.example.com/api/v4/...` |
| `CI_SERVER_HOST` | `gitlab.com` | `your-gitlab.example.com` |
| Package Registry | Always enabled | Must be enabled in Admin → Settings → Packages |
| npm scope | Must match top-level group | Same |
| `gitlabUrl` in `.releaserc.json` | `https://gitlab.com` | `https://your-gitlab.example.com` |

---

## 7. Troubleshooting

### 7.1 — 401 Unauthorized on `npm publish`

**Symptoms:** `npm ERR! 401 Unauthorized` during the release job.

**Causes & Fixes:**

| Cause | Fix |
|-------|-----|
| `.npmrc` not generated correctly | Check the `before_script` in the release job. Verify the `echo` commands produce valid `.npmrc` |
| Wrong project ID in `publishConfig` | Go to Settings → General → find numeric Project ID |
| Scope mismatch | `@my-group` in package name MUST match your GitLab top-level namespace (case-insensitive) |
| `CI_JOB_TOKEN` not available | Only available in CI jobs. Cannot test locally. |

**Debug command (add to pipeline):**
```yaml
- cat .npmrc
- npm whoami --registry=${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/packages/npm/
```

### 7.2 — 403 Forbidden on `npm publish`

**Symptoms:** `npm ERR! 403 Forbidden`

**Causes & Fixes:**

| Cause | Fix |
|-------|-----|
| Package scope doesn't match group | `@my-group` must exactly match the top-level GitLab group name |
| Packages feature disabled | Project → Settings → General → Visibility → Packages: Enabled |
| Token lacks write scope | `CI_JOB_TOKEN` has it by default; for PAT ensure `api` scope |
| Protected branch required | Some setups require the pipeline to run on a protected branch — `master` should already be protected |

### 7.3 — Version Already Exists (409 Conflict)

**Symptoms:** `npm ERR! 409 Conflict - @my-group/pid-components@1.2.3 already exists`

**Causes & Fixes:**

| Cause | Fix |
|-------|-----|
| Pipeline re-run after successful release | semantic-release detects existing tag and skips. But if the git plugin failed mid-way, the npm package may exist while the tag doesn't. **Fix:** manually create the git tag `v1.2.3` pointing to the release commit, then re-run. |
| Manual version bump committed | Never edit `version` in `package.json`. Reset it to `"0.0.0-semantically-released"` and let semantic-release handle it. |
| Duplicate tag | Delete the tag in Repository → Tags (if it was created incorrectly), then re-run. |

**Prevention:** The `[skip ci]` in the `@semantic-release/git` commit message prevents re-trigger. If you use squash merges, this is already handled.

### 7.4 — Tag Creation Fails

**Symptoms:** `semantic-release: An error occurred while running the publish step` or `remote: You are not allowed to create this tag`

**Causes & Fixes:**

| Cause | Fix |
|-------|-----|
| `GITLAB_TOKEN` not set or wrong | Check Settings → CI/CD → Variables. Must be Masked + Protected. |
| Token doesn't have `write_repository` | Recreate the Project Access Token with correct scopes |
| Protected tags block the token | Go to Settings → Repository → Protected Tags. Pattern `v*` must allow the token's role (Maintainer). |
| Token expired | Rotate the Project Access Token |
| `git remote` URL wrong | Verify the `git remote set-url` command uses `GITLAB_TOKEN`, not `CI_JOB_TOKEN` |

### 7.5 — Package Install Fails from Other Projects

**Symptoms:** Consumer runs `npm install @my-group/pid-components` and gets 404 or 401.

**Causes & Fixes:**

| Cause | Fix |
|-------|-----|
| No `.npmrc` in consumer project | Create `.npmrc` with the group-level registry endpoint and auth token |
| Wrong registry URL | Use the **group** endpoint (`/groups/GROUP_ID/-/packages/npm/`) not the project endpoint for cross-project access |
| Token lacks `read_package_registry` | Use a Deploy Token with `read_package_registry` scope |
| Package visibility is Private | Either make it internal/public, or ensure the consumer's token has access to the project |
| Group ID vs Project ID confusion | Group endpoint uses Group ID (numeric). Find it on the group's main page. |

**Debug (consumer side):**
```bash
# Verify registry config
npm config get @my-group:registry

# Test auth
npm whoami --registry=https://gitlab.com/api/v4/groups/YOUR_GROUP_ID/-/packages/npm/

# Verbose install
npm install @my-group/pid-components --verbose
```

### 7.6 — semantic-release Finds No Commits / No Release

**Symptoms:** Pipeline succeeds but no version is published. Log says "No release published."

**Causes & Fixes:**

| Cause | Fix |
|-------|-----|
| No conventional commits since last tag | Commit messages must start with `feat:`, `fix:`, etc. A `chore:` commit won't trigger a release. |
| Shallow clone | GitLab CI default clone depth may be too shallow. Add `GIT_DEPTH: 0` to variables or set `GIT_STRATEGY: clone`. |
| Wrong branch config | Ensure `.releaserc.json` has `"branches": ["master"]` matching your actual default branch name |

**Add to `.gitlab-ci.yml` variables:**
```yaml
variables:
  GIT_DEPTH: 0    # Full clone for semantic-release to analyze all commits
```

### 7.7 — CHANGELOG.md Commit Causes Infinite Pipeline Loop

**Symptoms:** Release pipeline triggers another pipeline, which triggers another...

**Fix:** The `[skip ci]` in the `@semantic-release/git` message config handles this:
```json
"message": "chore(release): ${nextRelease.version} [skip ci]"
```

If it still loops, add a pipeline rule:
```yaml
workflow:
  rules:
    - if: $CI_COMMIT_MESSAGE =~ /\[skip ci\]/
      when: never
    - when: always
```

---

## Quick Reference: All Files Checklist

```
✅ .gitlab-ci.yml          — Pipeline definition
✅ .releaserc.json          — semantic-release config
✅ package.json             — name, publishConfig, repository, devDeps
✅ vite.config.ts           — Library build mode
✅ commitlint.config.js     — (Optional) Enforce commit format

Do NOT commit:
❌ .npmrc                   — Generated at CI runtime

GitLab Settings:
✅ Protected branch: master
✅ Protected tags: v*
✅ CI/CD Variable: GITLAB_TOKEN (masked, protected)
✅ Packages feature: enabled
```
