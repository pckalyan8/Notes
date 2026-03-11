# Phase 8 — Package Managers & Module Systems
## Frontend Angular Mastery Roadmap
> **Duration:** 1 week | **Goal:** Manage dependencies professionally

---

## Table of Contents

1. [8.1 — npm Deep Dive](#81--npm-deep-dive)
2. [8.2 — Yarn & pnpm](#82--yarn--pnpm)
3. [8.3 — Module Systems](#83--module-systems)
4. [8.4 — Dependency Management Best Practices](#84--dependency-management-best-practices)
5. [Summary & Key Takeaways](#summary--key-takeaways)

---

## 8.1 — npm Deep Dive

### What Is npm and Why Does It Exist?

**npm** (Node Package Manager) is the world's largest software registry and the default package manager that ships with Node.js. It solves one of the most fundamental problems in software development: **dependency management** — the ability to declare that your project needs specific external libraries, and to install, update, and remove them reliably and reproducibly across every developer's machine and every CI server.

Before npm (and before package managers in general), sharing JavaScript code meant manually downloading files, copying them into your project, and hoping they worked. There was no way to declare "I need version 17 of Angular", no way to automatically pull in Angular's own dependencies, and no way to ensure two developers on the same team had identical setups. npm changed all of this in 2010, and today the npm registry hosts over 2.5 million packages — the largest collection of open-source libraries in the world.

npm does three distinct things:
1. **Registry** — a giant online database of public packages at `registry.npmjs.org`
2. **CLI tool** — the `npm` command you run in your terminal
3. **Package format** — the conventions (`package.json`, `node_modules/`) that define what a package is

---

### package.json — The Heart of Every Project

Every npm project has a `package.json` file in its root directory. This file is the manifest — it describes the project, declares its dependencies, defines scripts, and controls countless aspects of how tools interact with it.

```json
{
  "name": "my-angular-app",
  "version": "1.0.0",
  "description": "A production-grade Angular application",
  "private": true,
  "license": "MIT",
  "author": {
    "name": "Your Name",
    "email": "you@example.com"
  },
  "engines": {
    "node": ">=22.0.0",
    "npm": ">=10.0.0"
  },
  "scripts": {
    "start": "ng serve",
    "build": "ng build",
    "build:prod": "ng build --configuration production",
    "test": "ng test",
    "test:ci": "ng test --watch=false --browsers=ChromeHeadless",
    "lint": "ng lint",
    "format": "prettier --write \"src/**/*.{ts,html,scss,json}\"",
    "format:check": "prettier --check \"src/**/*.{ts,html,scss,json}\"",
    "prepare": "husky"
  },
  "dependencies": {
    "@angular/animations": "^21.0.0",
    "@angular/common": "^21.0.0",
    "@angular/compiler": "^21.0.0",
    "@angular/core": "^21.0.0",
    "@angular/forms": "^21.0.0",
    "@angular/platform-browser": "^21.0.0",
    "@angular/platform-browser-dynamic": "^21.0.0",
    "@angular/router": "^21.0.0",
    "rxjs": "~7.8.0",
    "tslib": "^2.6.0",
    "zone.js": "~0.15.0"
  },
  "devDependencies": {
    "@angular-devkit/build-angular": "^21.0.0",
    "@angular/cli": "^21.0.0",
    "@angular/compiler-cli": "^21.0.0",
    "@types/jasmine": "~5.1.0",
    "husky": "^9.0.0",
    "lint-staged": "^15.0.0",
    "prettier": "^3.0.0",
    "typescript": "~5.9.0"
  },
  "peerDependencies": {},
  "optionalDependencies": {}
}
```

---

### Dependency Types Explained

Understanding which of the four dependency fields to use is fundamental. Getting it wrong creates either bloated production bundles (putting dev tools in `dependencies`) or broken installations (missing required peers).

**`dependencies`** — packages your application needs to run in production. The Angular framework packages (`@angular/core`, `@angular/common`, etc.), RxJS, and any utility libraries you actually use in your application code go here. When someone installs your package, npm will also install everything in `dependencies`.

**`devDependencies`** — packages needed only during development, testing, and building — never in production. The Angular CLI, TypeScript compiler, ESLint, Prettier, Jest, Karma, Husky — all of these go here. `npm install --omit=dev` (or the older `--production`) skips these, which is exactly what you want in a Docker production image.

```bash
# Install a runtime dependency
npm install @angular/material

# Install a development-only dependency
npm install --save-dev @types/node

# Shorthand
npm i rxjs          # → dependencies
npm i -D typescript # → devDependencies
```

**`peerDependencies`** — packages that your package *requires the consumer to provide*, but which your package does not install itself. This is the mechanism used by Angular libraries. When you publish `@my-org/my-angular-lib`, you declare `@angular/core` as a peer dependency — meaning "whoever installs my library must already have Angular installed; I won't install a second copy of Angular for myself." This prevents duplicate framework copies in the final bundle.

```json
// package.json of an Angular component library
{
  "name": "@my-org/my-angular-lib",
  "peerDependencies": {
    "@angular/core": ">=18.0.0 <22.0.0",
    "@angular/common": ">=18.0.0 <22.0.0"
  }
}
```

When a peer dependency version is not satisfied (e.g., the consumer has Angular 17 but your library requires Angular 18+), npm prints a warning. In npm v7+, peer dependencies are automatically installed by default, but mismatches still generate warnings.

**`optionalDependencies`** — packages that enhance functionality when available but whose absence should not fail the installation. If an optional dependency fails to install (e.g., a platform-specific native module that does not compile on the current OS), npm continues without error. Use this sparingly — it is rare in frontend work.

---

### Semantic Versioning (SemVer) — The Contract Between Packages

Every npm package version follows the **Semantic Versioning** specification: `MAJOR.MINOR.PATCH` (e.g., `17.3.2`).

- **MAJOR** — incremented when you make **breaking changes** (changes that require consumers to update their code to stay compatible). Angular increments major version every 6 months.
- **MINOR** — incremented when you add **new features in a backwards-compatible way**. Existing code continues to work without changes.
- **PATCH** — incremented for **backwards-compatible bug fixes** only. No new features, no breaking changes.

There are also pre-release versions: `18.0.0-alpha.1`, `18.0.0-beta.3`, `18.0.0-rc.2`. These are published before the stable release for early testing.

**Version range specifiers** in `package.json` tell npm which versions it is allowed to install:

```
Exact version:   "17.3.2"         → only this exact version
Caret (^):       "^17.3.2"        → >=17.3.2 and <18.0.0   (allows minor + patch upgrades)
Tilde (~):       "~17.3.2"        → >=17.3.2 and <17.4.0   (allows only patch upgrades)
Greater than:    ">17.0.0"        → any version above 17.0.0
Range:           ">=17.0.0 <18"   → explicit range
Latest:          "*"              → any version (dangerous — never use in production)
```

**The caret (`^`) is the default** applied when you run `npm install some-package`. It means "I trust this package to follow SemVer — give me the latest compatible minor version." This is usually correct, but for packages with a history of accidental breaking changes in minors, pin to an exact version.

**Why `~` for rxjs and zone.js in Angular projects?** Angular's `package.json` uses `~7.8.0` for rxjs. The tilde restricts updates to patch versions only. This is more conservative — used for packages where even minor version bumps sometimes introduce subtle behaviour changes that affect Angular's internal use of the library.

```json
// In Angular project — typical version strategies:
"@angular/core": "^21.0.0",   // ^ — trust Angular's SemVer, allow minor updates
"rxjs": "~7.8.0",             // ~ — conservative, patch updates only
"typescript": "~5.9.0"        // ~ — TypeScript minor versions can have breaking changes
```

> **Best Practice:** Never use `*` or `latest` as version ranges in `package.json`. Always commit your `package-lock.json`. Use `^` for most dependencies, `~` for lower-level infrastructure packages (TypeScript, rxjs, zone.js). If a package is known to break SemVer frequently, pin to an exact version.

---

### package-lock.json — Deterministic Installs

`package-lock.json` is one of the most important and most misunderstood files in a Node.js project.

**The problem it solves:** `package.json` says "I need Angular version `^21.0.0`". This allows npm to install `21.0.0`, `21.1.0`, or `21.2.0` — whichever is latest when you run `npm install`. If you run `npm install` today you might get `21.0.0`. A colleague running it next week might get `21.1.0`. Your CI server might have a cached version from last month. These differences can cause subtle bugs that are nearly impossible to diagnose.

**The solution:** `package-lock.json` records the **exact version of every package** (and every package's dependency, and so on recursively) that was installed at a specific point in time. It locks the entire dependency tree. When npm sees a `package-lock.json`, it installs exactly those versions — regardless of what newer versions might be available.

```json
// package-lock.json — excerpt showing the locked tree
{
  "name": "my-angular-app",
  "version": "1.0.0",
  "lockfileVersion": 3,
  "requires": true,
  "packages": {
    "": {
      "dependencies": {
        "@angular/core": "^21.0.0"
      }
    },
    "node_modules/@angular/core": {
      "version": "21.0.2",           // exact version pinned here
      "resolved": "https://registry.npmjs.org/@angular/core/-/core-21.0.2.tgz",
      "integrity": "sha512-...",     // cryptographic hash to verify integrity
      "dependencies": {
        "tslib": "^2.6.0"
      },
      "peerDependencies": {
        "rxjs": "^7.4.0"
      }
    }
  }
}
```

**Key rule: Always commit `package-lock.json` to your repository.** It is not a generated artifact to be ignored — it is a precise specification of your dependency tree that guarantees every developer, every CI run, and every production deployment installs identical dependencies.

---

### npm install vs npm ci

There are two commands for installing dependencies, and choosing the wrong one is a common mistake:

```bash
# npm install — for development use
npm install
# - Reads package.json
# - Resolves the best compatible versions
# - UPDATES package-lock.json if needed
# - Installs to node_modules/

# npm ci — for CI/CD and production deployments
npm ci
# - Reads package-lock.json ONLY (ignores package.json for versions)
# - FAILS if package-lock.json is out of sync with package.json
# - Deletes node_modules/ first, then installs fresh
# - Never updates package-lock.json
# - Significantly faster than npm install in CI environments
```

**Rule of thumb:**
- Developer workstation, first setup → `npm install`
- After someone changes `package.json` on your team → `npm install`
- CI/CD pipeline → always `npm ci`
- Production Docker builds → always `npm ci`

> **Best Practice:** In your GitHub Actions workflow, always use `npm ci`, not `npm install`. `npm ci` is faster (skips resolution), more reliable (exact versions), and will catch the error of a package-lock.json that is out of sync with package.json (which would cause mysterious behaviour in production).

---

### npm install, update, audit, outdated

```bash
# Install all dependencies (from package-lock.json)
npm install
npm ci                    # for CI (clean install)

# Add a new package (updates both package.json and package-lock.json)
npm install lodash
npm install --save-dev vitest

# Remove a package
npm uninstall lodash

# Update a specific package to the latest version allowed by its range in package.json
npm update @angular/core

# Upgrade a package BEYOND its current range (use with caution)
npm install @angular/core@latest

# See which installed packages are outdated (have newer versions available)
npm outdated

# Output:
# Package          Current  Wanted  Latest  Location
# @angular/core    21.0.0   21.0.3  21.0.3  node_modules/@angular/core
# lodash           4.17.20  4.17.21 4.17.21 node_modules/lodash

# Check for security vulnerabilities in your dependency tree
npm audit

# Automatically fix vulnerabilities where possible (changes package-lock.json)
npm audit fix

# Fix even if it requires a semver-major version bump (more aggressive — review output)
npm audit fix --force
```

---

### npm Scripts and Lifecycle Hooks

The `scripts` field in `package.json` is a powerful task runner. Any key you define becomes runnable with `npm run <key>`. Some script names are special **lifecycle hooks** that npm runs automatically at specific moments.

**Custom scripts:**

```json
{
  "scripts": {
    "start":        "ng serve",
    "build":        "ng build",
    "test":         "ng test",
    "lint":         "ng lint",
    "analyze":      "ng build --stats-json && npx webpack-bundle-analyzer dist/stats.json",
    "storybook":    "storybook dev -p 6006",
    "e2e":          "playwright test"
  }
}
```

```bash
npm start           # shortcut for npm run start (special alias)
npm test            # shortcut for npm run test (special alias)
npm run build       # must use "run" for custom scripts
npm run analyze     # runs multiple commands sequentially with &&
```

**Lifecycle hooks — automatic execution:**

npm automatically runs scripts with `pre` and `post` prefixes around the matching script name. This is how you add setup and teardown steps around any script:

```json
{
  "scripts": {
    "prebuild":      "echo 'Cleaning dist...' && rm -rf dist",
    "build":         "ng build --configuration production",
    "postbuild":     "echo 'Build complete. Uploading source maps...' && node scripts/upload-sourcemaps.js",

    "pretest":       "ng lint --quiet",
    "test":          "ng test --watch=false",
    "posttest":      "node scripts/send-coverage-report.js",

    "prepare":       "husky",   // runs after npm install — used to set up git hooks
    "prepublishOnly":"npm test", // runs before publishing to npm registry

    "preinstall":    "node check-node-version.js"  // runs before any npm install
  }
}
```

**Built-in lifecycle hook order for `npm publish`:**
```
prepublish → prepare → prepublishOnly → prepack → pack → postpack → publish → postpublish
```

**Built-in lifecycle hook order for `npm install`:**
```
preinstall → install → postinstall → prepublish → prepare
```

The `prepare` hook is especially important — it runs both after `npm install` and before `npm publish`, making it the standard place to set up git hooks with Husky.

**Composing scripts with `&&`, `|`, and `npm-run-all`:**

```json
{
  "scripts": {
    // Run sequentially — second only runs if first succeeds
    "validate": "npm run lint && npm run test && npm run build",

    // Run in parallel using npm-run-all (cross-platform tool)
    "dev": "npm-run-all --parallel start:api start:app",
    "start:api": "cd api && npm start",
    "start:app": "ng serve"
  }
}
```

---

### npx — Execute Without Installing

`npx` (included with npm 5.2+) lets you run a package's CLI without globally installing it. This is crucial for keeping your machine clean and ensuring everyone uses the same version of CLI tools.

```bash
# Run Angular CLI without globally installing it
npx @angular/cli new my-app
npx ng generate component my-component

# Run a one-time code migration
npx @angular/core:standalone-migration

# Run a specific version of a tool (not necessarily what's installed)
npx typescript@5.9 tsc --version

# Run a local (project-level) package binary
# (same as ./node_modules/.bin/prettier)
npx prettier --write "src/**/*.ts"

# Run a remote script without installing anything persistently
# (use with caution — verify the package source first)
npx create-nx-workspace@latest my-nx-app
```

**`npx` package resolution order:**
1. Check `./node_modules/.bin/` for a locally installed binary
2. If not found, temporarily download from the npm registry, run, then delete
3. Never permanently installs to the global scope

> **Best Practice:** Prefer `npx` and locally-installed packages over global installs for all project tooling. Global installs (`npm install -g @angular/cli`) cause version conflicts between projects and make your environment hard to reproduce. Use `npx` or add the tool to `devDependencies`.

---

### npm Workspaces

npm Workspaces (available since npm 7) are npm's built-in solution for managing monorepos — a single repository containing multiple related packages.

```json
// Root package.json (workspace root)
{
  "name": "my-monorepo",
  "private": true,
  "workspaces": [
    "apps/*",
    "libs/*"
  ]
}
```

```
my-monorepo/
├── package.json          ← workspace root (defines workspaces)
├── package-lock.json     ← single lockfile for the ENTIRE monorepo
├── node_modules/         ← single shared node_modules/ for all packages
├── apps/
│   ├── web-app/
│   │   └── package.json  ← { "name": "@myorg/web-app" }
│   └── admin-app/
│       └── package.json  ← { "name": "@myorg/admin-app" }
└── libs/
    ├── ui-components/
    │   └── package.json  ← { "name": "@myorg/ui-components" }
    └── shared-utils/
        └── package.json  ← { "name": "@myorg/shared-utils" }
```

```bash
# Install ALL dependencies for all workspaces (run from root)
npm install

# Run a script in a specific workspace
npm run build --workspace=apps/web-app

# Run a script in ALL workspaces
npm run test --workspaces

# Add a dependency to a specific workspace
npm install lodash --workspace=apps/web-app

# Add a workspace package as a dependency of another workspace
# (npm automatically links them via symlinks — no local publishing needed)
npm install @myorg/shared-utils --workspace=apps/web-app
```

One of the key benefits of workspaces is that packages within the monorepo can depend on each other directly. When `@myorg/web-app` depends on `@myorg/shared-utils`, npm creates a symlink in `node_modules/@myorg/shared-utils` pointing to `libs/shared-utils/`. Changes to the library are immediately reflected in the app without any publishing or linking step.

> **Important:** For serious monorepo work in Angular, use **Nx** (which builds on top of npm/yarn/pnpm workspaces) rather than raw npm workspaces. Nx adds project graph analysis, affected build detection, remote caching, and generator scaffolding that raw workspaces do not provide. But understanding npm workspaces is the prerequisite for understanding how Nx works under the hood.

---

### .npmrc — npm Configuration File

`.npmrc` is the configuration file for npm. It can exist at the project level (committed to the repo), user level (`~/.npmrc`), or global level. Project-level `.npmrc` is committed to the repository and controls how npm behaves for that project.

```ini
# .npmrc — project-level configuration (commit this file)

# Use a specific registry for all packages (default is registry.npmjs.org)
registry=https://registry.npmjs.org/

# Use a private registry for packages in the @myorg scope
@myorg:registry=https://npm.pkg.github.com/

# Set exact versions when installing (avoids ^ prefix)
save-exact=true

# Disable running post-install scripts (security consideration for CIs)
ignore-scripts=false

# Force the specific Node.js engine version to be respected
engine-strict=true

# Prevent the package-lock.json from being created (rare — use only for libraries)
# package-lock=false

# Cache location (useful to point to a shared cache in CI)
# cache=/home/runner/.npm
```

```ini
# ~/.npmrc — user-level configuration (NOT committed)

# Authentication token for npm registry publishing
//registry.npmjs.org/:_authToken=${NPM_TOKEN}

# Authentication for GitHub Packages
//npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}
```

> **Important:** Never commit authentication tokens directly in `.npmrc`. Use environment variable interpolation (`${NPM_TOKEN}`) and inject the actual values through CI/CD secrets.

---

### Publishing Packages to the npm Registry

Publishing your own package to npm makes it reusable across projects and shareable with the community.

```bash
# Step 1: Authenticate
npm login
# (prompts for username, password, email, OTP)

# Step 2: Verify your package
npm pack --dry-run
# Shows exactly which files would be published WITHOUT actually publishing
# Review the output carefully — make sure you're not accidentally including secrets

# Step 3: Publish
npm publish

# Publish a pre-release version (will NOT be installed by default with npm install)
npm publish --tag beta

# Publish a scoped package as public (scoped packages are private by default)
npm publish --access public

# Step 4: Deprecate an old version (warn consumers without removing it)
npm deprecate my-package@1.0.0 "Please upgrade to 2.0.0 — critical security fix"
```

**What gets published — the `.npmignore` file:**

By default, npm publishes everything except what is in `.gitignore`. Add an `.npmignore` to exclude files that developers need but consumers don't (test files, source TypeScript before compilation, etc.):

```
# .npmignore
src/                # Publish only the compiled output, not TypeScript source
*.spec.ts           # Test files
*.spec.js
e2e/
.github/
cypress/
.env
```

**The `files` field in package.json (preferred over .npmignore):**

```json
{
  "name": "@myorg/my-lib",
  "version": "1.0.0",
  "main": "dist/index.js",
  "module": "dist/esm/index.js",
  "types": "dist/index.d.ts",
  "files": [
    "dist/",
    "README.md",
    "LICENSE"
  ]
}
```

The `files` allowlist is more explicit and safer than `.npmignore` — it is impossible to accidentally publish something you didn't intend to include.

---

### Private Registries: Verdaccio and Artifactory

Large organisations need a private npm registry for:
- **Proprietary packages** that should not be published publicly
- **Caching public packages** (for security auditing, offline access, and speed)
- **First-party packages** shared between teams but not meant for public consumption

**Verdaccio** is a lightweight, open-source private npm registry you can self-host:

```bash
# Install and run Verdaccio locally
npm install -g verdaccio
verdaccio
# → Runs at http://localhost:4873

# Configure your project to use it for a specific scope
npm config set @myorg:registry http://localhost:4873/

# Publish to your private registry
npm publish --registry http://localhost:4873/
```

**Artifactory** (by JFrog) and **GitHub Packages** are enterprise-grade alternatives with access control, audit logging, and vulnerability scanning. Most large companies use one of these.

```ini
# .npmrc for GitHub Packages
@myorg:registry=https://npm.pkg.github.com/
//npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}
```

---

## 8.2 — Yarn & pnpm

### Why Other Package Managers Exist

npm dominated the Node.js ecosystem from the start, but it had significant problems in its early years: slow installation speeds (sequential installation, no parallelism), a non-deterministic install algorithm (the same `package.json` could produce different `node_modules/` structures on different machines), and security concerns (npm scripts running arbitrary code on install). These problems drove Facebook to create Yarn in 2016 and motivated the development of pnpm.

Understanding Yarn and pnpm is important because: many Angular projects and open-source repos use them, Nx officially supports all three, and each has trade-offs worth understanding for your team's choice.

---

### Yarn Classic (v1) — The Original Alternative

Yarn Classic introduced three concepts that were later adopted by npm itself:

1. **`yarn.lock`** — a deterministic lockfile (npm later created `package-lock.json`)
2. **Parallel installation** — all packages download simultaneously
3. **Offline mode** — packages are cached locally and can be installed without internet

```bash
# Installing Yarn Classic (still widely used, but in maintenance mode)
npm install -g yarn

# Basic commands (similar to npm but with different syntax)
yarn                    # install all dependencies (reads yarn.lock)
yarn add lodash         # add to dependencies
yarn add --dev vitest   # add to devDependencies
yarn remove lodash      # remove package
yarn upgrade lodash     # upgrade to latest allowed version
yarn outdated           # check for outdated packages

# Run scripts (no "run" keyword needed for custom scripts)
yarn build
yarn test
yarn start

# yarn.lock — the equivalent of package-lock.json
# Always commit yarn.lock to version control
```

**Yarn Workspaces** (introduced before npm Workspaces) work similarly:

```json
// Root package.json
{
  "private": true,
  "workspaces": ["apps/*", "packages/*"]
}
```

```bash
yarn workspace @myorg/web-app add lodash   # add dep to specific workspace
yarn workspaces run build                  # run build script in all workspaces
```

---

### Yarn Berry (v2, v3, v4) — The Modern Rewrite

Yarn Berry is a complete rewrite of Yarn with a radically different architecture. Its most controversial feature is **Plug'n'Play (PnP)** — an alternative to the traditional `node_modules/` folder.

**The problem with `node_modules/`:** A standard Angular project has `node_modules/` folders containing hundreds of thousands of files totalling gigabytes of disk space. Installing dependencies takes minutes. Most CI systems spend a significant portion of build time on `npm ci`. The directory structure allows packages to accidentally rely on dependencies they didn't declare (phantom dependencies).

**Plug'n'Play (PnP) — how it works:** Instead of copying packages into `node_modules/`, Yarn Berry generates a single `.pnp.cjs` file — a resolver map that tells Node.js exactly where each package lives (in a compressed `.yarn/cache/` folder). No `node_modules/` is created at all. The `.yarn/cache/` can be committed directly to the repository, meaning `yarn install` becomes near-instantaneous on fresh clones (zero-installs).

```bash
# Create a new project with Yarn Berry
yarn set version berry

# The project now has:
# .yarn/
#   cache/          ← compressed package zips (can be committed)
#   releases/
#     yarn-4.x.x.cjs
# .pnp.cjs          ← the resolver map
# .yarnrc.yml       ← Yarn Berry configuration

# Install (generates .pnp.cjs — no node_modules/)
yarn install

# Add a package
yarn add @angular/material

# Run a script
yarn ng serve
```

**`.yarnrc.yml` configuration:**

```yaml
# .yarnrc.yml
nodeLinker: pnp          # Use Plug'n'Play (default in Berry)
# nodeLinker: node-modules  # Fall back to classic node_modules/ if tools are incompatible

yarnPath: .yarn/releases/yarn-4.1.1.cjs

packageExtensions:       # Fix broken peer dependencies in third-party packages
  "react-dom@*":
    peerDependencies:
      react: "*"
```

**Challenges with Yarn PnP and Angular:**

PnP is powerful but has compatibility challenges. Many tools (including some Angular-adjacent packages) assume `node_modules/` exists and hardcode paths to it. When using Yarn Berry with Angular:

- Use `nodeLinker: node-modules` in `.yarnrc.yml` for maximum compatibility (gives up PnP's benefits but keeps Yarn's other features)
- Or use PnP with the `@yarnpkg/pnpify` tool to patch incompatible tools

In practice, most Angular teams using Yarn Berry either use `nodeLinker: node-modules` mode or switch to pnpm (described next), which achieves similar disk efficiency without abandoning `node_modules/`.

---

### pnpm — The Performance-Focused Manager

**pnpm** (performant npm) solves the `node_modules/` bloat problem with a different and arguably more elegant approach than Yarn PnP: a **global content-addressable store** combined with **hard links and symlinks**.

**How the global store works:**

When you install a package, pnpm downloads it to a global store on your machine (typically `~/.pnpm-store/`). Every file in every version of every package lives here. When you install the same package version in a second project, pnpm does not download it again — it creates **hard links** from the project's `node_modules/` to the files in the global store. This means:

1. A package installed in 10 projects takes disk space only once (hard links, not copies)
2. Installation of already-seen packages is nearly instant (just creates links)
3. `node_modules/` is still present (full tool compatibility), but takes far less disk space

```
Global store (~/.pnpm-store/):
├── v3/
│   └── files/
│       ├── 00/abc123...    ← actual file content (by hash)
│       ├── 01/def456...
│       └── ...

Project node_modules/:
├── lodash/
│   └── lodash.js           → hard link to ~/.pnpm-store/v3/files/...
└── .modules.yaml
```

**Non-flat node_modules/ structure:**

pnpm enforces strict dependency declarations by making `node_modules/` non-flat. If your code imports a package you didn't declare in `package.json`, Node.js will throw a "Cannot find module" error — even if that package is installed (because another package depends on it). This prevents phantom dependencies — a class of bugs that npm's flat `node_modules/` allows.

```bash
# Installing pnpm
npm install -g pnpm
# or (recommended — avoids global install):
corepack enable  # Node.js 16.9+ built-in
corepack use pnpm@latest

# pnpm commands (very similar to npm)
pnpm install              # install all dependencies
pnpm add lodash           # add to dependencies
pnpm add -D vitest        # add to devDependencies
pnpm remove lodash        # remove
pnpm update               # update to latest allowed versions
pnpm outdated             # check for outdated packages
pnpm audit                # security audit

# Running scripts
pnpm run build
pnpm build                # "run" is optional in pnpm

# pnpm-lock.yaml — pnpm's lockfile (commit this)
```

**pnpm Workspaces:**

```yaml
# pnpm-workspace.yaml (replaces workspaces field in package.json)
packages:
  - 'apps/*'
  - 'libs/*'
  - '!**/__tests__/**'   # Exclude test directories
```

```bash
# Run a command in a specific workspace
pnpm --filter @myorg/web-app build

# Run in all workspaces matching a pattern
pnpm --filter "@myorg/*" test

# Add a dependency to a specific workspace
pnpm --filter @myorg/web-app add lodash

# Link a workspace package as a dependency of another
pnpm --filter @myorg/web-app add @myorg/shared-utils
```

---

### Lockfile Comparison

Understanding what each package manager's lockfile contains helps you reason about reproducibility and migration.

| Feature | npm | Yarn Classic | Yarn Berry | pnpm |
|---|---|---|---|---|
| Lockfile name | `package-lock.json` | `yarn.lock` | `yarn.lock` | `pnpm-lock.yaml` |
| Format | JSON | Custom format | Custom format | YAML |
| Lockfile version | v3 (npm 7+) | — | — | — |
| Records exact versions | ✅ | ✅ | ✅ | ✅ |
| Records integrity hashes | ✅ | ✅ | ✅ | ✅ |
| Human readable | Moderate | Yes | Yes | Yes |
| Commit to repo | Always | Always | Always | Always |

All three lockfiles accomplish the same fundamental goal: record exact package versions so installs are reproducible. The format differences matter only for tooling that parses them (like Renovate and Dependabot, which support all three).

---

### Choosing Between npm, Yarn, and pnpm

There is no universally correct answer — the right choice depends on your team's context, existing infrastructure, and priorities.

**Choose npm when:**
- You want zero additional tooling — it ships with Node.js
- Your team includes developers new to frontend tooling
- You are starting a fresh Angular project without complex monorepo needs
- Compatibility with all tools is the priority (npm has the broadest support)
- You use GitHub Actions (excellent built-in npm caching support)

**Choose Yarn Classic (v1) when:**
- You are maintaining an existing project that already uses it
- You want workspaces with slightly more mature tooling than early npm workspaces

**Choose Yarn Berry (v2+) when:**
- You want zero-install CI (commit cache, near-instant installs on every run)
- Your team has addressed PnP compatibility for your specific Angular toolchain
- You want Yarn Berry's superior workspace protocol features

**Choose pnpm when:**
- Disk space is a concern (mono-repo machines, shared CI agents with many projects)
- You want strict phantom dependency prevention
- You are using Nx (pnpm is Nx's recommended package manager as of 2025)
- You want npm-compatible commands with better performance
- You are building a new monorepo from scratch

```bash
# Migration paths
# npm → pnpm: just swap commands, add pnpm-workspace.yaml, delete package-lock.json
# npm → Yarn Classic: yarn import (imports package-lock.json into yarn.lock)
# npm → Yarn Berry: yarn set version berry
```

> **Best Practice:** For new Angular projects in 2026, **pnpm** is increasingly the recommendation, especially with Nx. For teams that prefer the path of least resistance, **npm** with `npm ci` in CI is perfectly fine. Avoid Yarn Classic for new projects — it is in maintenance mode with no new features.

---

## 8.3 — Module Systems

### Why Module Systems Exist

In the early days of JavaScript (pre-2009), there was no concept of "importing" code from another file. Every script tag loaded into a global scope, and all variables from all scripts shared the same global namespace. Building large applications meant manually ordering hundreds of `<script>` tags, carefully ensuring each library loaded before the code that used it, and dealing with name collision bugs when two libraries chose the same global variable name.

Module systems solved this by giving each file its own private scope, a way to declare what it exports, and a way to import what it needs from other files. Understanding all the module systems in use today is essential for frontend developers because you encounter all of them: Node.js uses CommonJS, the browser uses ES Modules natively, and older libraries may use AMD or UMD.

---

### CommonJS (CJS) — The Node.js Standard

CommonJS was developed for server-side JavaScript (Node.js, 2009). Its defining characteristics are **synchronous loading** and a **dynamic `require()` function**.

```javascript
// math.js — exporting with CommonJS
function add(a, b) {
  return a + b;
}

function multiply(a, b) {
  return a * b;
}

// Named exports via module.exports object
module.exports = { add, multiply };

// Or export a single value as the default
module.exports = function(a, b) { return a + b; };
```

```javascript
// app.js — importing with CommonJS
const { add, multiply } = require('./math');
const express = require('express');  // package from node_modules

console.log(add(2, 3));      // 5
console.log(multiply(4, 5)); // 20

// Dynamic require — can load modules conditionally at runtime
const platform = process.platform;
const platformUtils = require(`./utils/${platform}`); // dynamic path

// require() is synchronous — it BLOCKS until the file is read and evaluated
// This is fine in Node.js (file system access), but terrible in the browser
```

**How CommonJS resolution works:**

When you call `require('./math')`, Node.js:
1. Resolves the path relative to the current file (`./math.js`, `./math/index.js`, etc.)
2. Checks if the module is already in the cache (Node.js caches modules after first load)
3. If not cached, reads and evaluates the file, caches the result, and returns `module.exports`

For non-relative paths like `require('lodash')`, Node.js searches:
1. `./node_modules/lodash`
2. `../node_modules/lodash`
3. `../../node_modules/lodash`
4. ... up to the file system root

**Why CommonJS cannot be tree-shaken:**

Tree shaking (dead code elimination) requires knowing at build time which exports are used. CommonJS `require()` is a runtime function call — it can be conditional, it can use a dynamic string, and `module.exports` can be mutated at any point. A bundler cannot statically analyse this:

```javascript
// This is valid CommonJS — but impossible to statically analyse
if (Math.random() > 0.5) {
  module.exports = require('./module-a');
} else {
  module.exports = require('./module-b');
}
```

**CommonJS in Angular context:** Node.js tools (Angular CLI, webpack config, Jest config files) use CommonJS. Application code uses ES Modules. When you see `.cjs` file extensions or `require()` in configuration files, that's CommonJS.

---

### AMD (Asynchronous Module Definition) — Historical Context

AMD was developed around 2009–2011 specifically to solve CommonJS's synchronous loading problem for **browsers**. The browser cannot synchronously load files from disk — it must make network requests, which are inherently asynchronous. AMD's `define()` function declares dependencies upfront and receives them in a callback when they're ready.

```javascript
// AMD module definition
define(['jquery', 'lodash'], function($, _) {
  // This function is called when jQuery and lodash are loaded
  function greet(name) {
    return _.capitalize(`hello, ${name}`);
  }
  return { greet };
});

// Using an AMD module
require(['my-module'], function(myModule) {
  console.log(myModule.greet('world')); // Hello, World
});
```

AMD was primarily used with the **RequireJS** library. It is now largely obsolete in frontend development — ES Modules have superseded it completely, and bundlers like webpack and esbuild handle async loading more efficiently through code splitting.

> **Why learn AMD?** You will encounter it in legacy codebases built before 2015, some older jQuery plugins, and library source code that still supports multiple module formats. Recognise it and know to convert it to ES Modules when modernising.

---

### UMD (Universal Module Definition) — Library Compatibility Format

UMD is not really a new module system — it is a **wrapper pattern** that makes a single file work as a CommonJS module, an AMD module, AND a global variable (for direct `<script>` tag usage). Library authors used it to maximise compatibility.

```javascript
// UMD wrapper (this is auto-generated by bundlers for libraries)
(function (root, factory) {
  if (typeof define === 'function' && define.amd) {
    // AMD environment (RequireJS)
    define(['jquery'], factory);
  } else if (typeof module === 'object' && module.exports) {
    // CommonJS environment (Node.js)
    module.exports = factory(require('jquery'));
  } else {
    // Browser global (script tag, no module system)
    root.myLibrary = factory(root.jQuery);
  }
}(typeof self !== 'undefined' ? self : this, function (jQuery) {
  // Actual library code
  function myLibrary() { /* ... */ }
  return myLibrary;
}));
```

**UMD in the wild:** Many older npm packages ship their CDN builds as UMD (lodash, moment.js, jQuery). When you include a library from a CDN in an `index.html` `<script>` tag without a module system, you're usually using a UMD bundle.

UMD is rarely written by hand today. Rollup (the bundler used for libraries) generates it automatically with `output: { format: 'umd' }`. For new libraries in 2026, you would publish ES Modules (ESM) and optionally CommonJS — UMD is only needed if you still need to support `<script>` tag usage.

---

### ES Modules (ESM) — The Modern Standard

ES Modules (also called ESM, ES6 Modules, or ECMAScript Modules) are the native module system standardised in ES2015. They are now supported in all modern browsers natively and in Node.js since v12. ESM is the module format Angular applications use exclusively.

**The defining characteristics of ES Modules:**

1. **Static structure** — all `import` and `export` statements are at the top level of a file, not inside functions or conditionals. This is what enables static analysis and tree shaking.
2. **Live bindings** — when you import a value, you get a live reference to the original export. If the exporting module changes the value of an export, your import reflects the change.
3. **Async loading** — the browser resolves and fetches module dependencies asynchronously and in parallel.
4. **Strict mode** — ES Modules are always in strict mode. No need for `'use strict';`.
5. **Own scope** — each module has its own scope. Top-level variables are not added to the global object.

```typescript
// math.ts — named exports
export function add(a: number, b: number): number {
  return a + b;
}

export function multiply(a: number, b: number): number {
  return a * b;
}

export const PI = 3.14159265358979;

// Default export (one per file — represents the "main thing" the module exports)
export default class Calculator {
  add = add;
  multiply = multiply;
}
```

```typescript
// app.ts — importing ES Modules

// Named imports — must match the exported names exactly
import { add, multiply, PI } from './math';

// Default import — name can be anything you choose
import Calculator from './math';

// Rename on import (useful to avoid naming conflicts)
import { add as mathAdd } from './math';

// Import everything as a namespace object
import * as MathUtils from './math';
console.log(MathUtils.add(2, 3));

// Import a package from node_modules (no relative path)
import { Injectable } from '@angular/core';
import { Observable } from 'rxjs';

// Import type only (erased at runtime — TypeScript 3.8+)
import type { User } from './user.model';

// Re-export (barrel files / index.ts pattern)
export { add, multiply } from './math';
export * from './string-utils';
export { default as Calculator } from './math';
```

**Dynamic imports — code splitting:**

`import()` is the dynamic, async version of `import`. It returns a Promise that resolves to the module. This is how Angular implements lazy loading of routes and `@defer` blocks.

```typescript
// Static import — always loaded upfront
import { HeavyComponent } from './heavy.component';

// Dynamic import — loaded on demand
async function loadHeavyFeature() {
  const { HeavyComponent } = await import('./heavy.component');
  // HeavyComponent is now available
}

// Angular's lazy route loading is built on dynamic import
const routes: Routes = [
  {
    path: 'admin',
    loadComponent: () => import('./admin/admin.component')
      .then(m => m.AdminComponent)
  }
];
```

**`import.meta`:**

`import.meta` is a special object available in ES Modules that provides metadata about the current module:

```typescript
// The URL of the current module file
console.log(import.meta.url);
// file:///Users/you/project/src/app/app.component.ts

// Used by Vite for environment variables
console.log(import.meta.env.MODE);         // 'development' or 'production'
console.log(import.meta.env.VITE_API_URL); // custom env variable

// Check if the module is the entry point
if (import.meta.url === import.meta.resolve('./main.ts')) {
  // This is the entry point
}
```

---

### Module Resolution — How Bundlers Find Files

When your code contains `import { Component } from '@angular/core'`, how does the bundler know where `@angular/core` lives? Module resolution is the algorithm used to map an import specifier to a physical file on disk.

**Node.js resolution algorithm (used by most bundlers):**

For **relative paths** (`./`, `../`):
1. Try the path as-is: `./utils` → look for `./utils.ts`, `./utils.js`
2. Try with extensions: `./utils.ts`, `./utils.tsx`, `./utils.js`, etc.
3. Try as directory with `index`: `./utils/index.ts`, `./utils/index.js`

For **bare specifiers** (package names like `@angular/core`, `lodash`):
1. Look in `./node_modules/@angular/core/`
2. Look in `../node_modules/@angular/core/`
3. Continue up the directory tree to the filesystem root
4. Once the package directory is found, read its `package.json` to determine the entry point (the `main`, `module`, or `exports` field)

```json
// @angular/core/package.json — entry points
{
  "name": "@angular/core",
  "exports": {
    ".": {
      "types": "./index.d.ts",
      "esm2022": "./esm2022/core.mjs",    // ES Module for bundlers
      "esm": "./esm2022/core.mjs",
      "default": "./fesm2022/core.mjs"
    },
    "./rxjs-interop": {
      "types": "./rxjs-interop/index.d.ts",
      "default": "./fesm2022/rxjs-interop.mjs"
    }
  }
}
```

**TypeScript path aliases** (configured in `tsconfig.json`):

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@app/*":     ["src/app/*"],
      "@env/*":     ["src/environments/*"],
      "@shared/*":  ["src/app/shared/*"],
      "@core/*":    ["src/app/core/*"]
    }
  }
}
```

```typescript
// Instead of painful relative paths:
import { UserService } from '../../../core/services/user.service';

// Use clean aliases:
import { UserService } from '@core/services/user.service';
import { environment } from '@env/environment';
```

Angular CLI (via esbuild) and webpack both read `tsconfig.json` path aliases and apply them during bundling — no additional bundler configuration needed.

---

### Dual CJS/ESM Packages

Many npm packages need to support both CommonJS (for Node.js tools and older bundlers) and ES Modules (for modern bundlers and tree shaking). This is called a **dual package**.

The modern way to publish a dual package is with the `exports` field in `package.json`:

```json
// my-library/package.json
{
  "name": "my-library",
  "version": "1.0.0",
  "exports": {
    ".": {
      "import": "./dist/esm/index.js",   // used when: import { x } from 'my-library'
      "require": "./dist/cjs/index.js",  // used when: require('my-library')
      "types": "./dist/types/index.d.ts" // TypeScript type definitions
    },
    "./utils": {
      "import": "./dist/esm/utils.js",
      "require": "./dist/cjs/utils.js",
      "types": "./dist/types/utils.d.ts"
    }
  },
  "main": "./dist/cjs/index.js",    // fallback for very old tools
  "module": "./dist/esm/index.js",  // read by some older bundlers (non-standard)
  "types": "./dist/types/index.d.ts"
}
```

**The dual package hazard:** When a package is loaded both as CJS and ESM in the same application (possible in complex setups), two separate instances of the module are created. This breaks Singletons and `instanceof` checks. The solution is to ensure a package is only ever loaded in one format per application — bundler configuration usually handles this automatically.

**`.mjs` and `.cjs` file extensions:** Node.js uses file extensions to determine module type:
- `.mjs` → always treated as ES Module
- `.cjs` → always treated as CommonJS
- `.js` → treated as the type declared in the nearest `package.json`'s `"type"` field
  - `"type": "module"` → `.js` files are ESM
  - `"type": "commonjs"` (default) → `.js` files are CJS

Angular libraries built with `ng-packagr` publish as pure ES Modules (`.mjs` files) following the Angular Package Format (APF) specification.

---

## 8.4 — Dependency Management Best Practices

### Keeping Dependencies Updated

Outdated dependencies are a silent risk. They accumulate security vulnerabilities, miss performance improvements, and eventually become so far behind that upgrading becomes a painful multi-day project. Regular, incremental updates prevent this.

```bash
# See all outdated packages
npm outdated

# Output:
# Package                Current  Wanted  Latest  Location
# @angular/core          20.0.0   20.2.3  21.0.0  node_modules/@angular/core
# @angular/cli           20.0.0   20.2.3  21.0.0  node_modules/@angular/cli
# eslint                  8.56.0   8.57.1   9.18.0  node_modules/eslint

# Update within the allowed range in package.json
npm update

# For major version upgrades (Angular, TypeScript), use ng update
npx ng update @angular/core @angular/cli

# Angular CLI handles migrations automatically on major upgrades
# It runs schematic-based migrations that update your code to the new API
```

**Automating dependency updates — Dependabot and Renovate:**

Manual monitoring of outdated packages does not scale. Set up automated tools:

```yaml
# .github/dependabot.yml — GitHub's Dependabot
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "09:00"
    groups:
      # Group all Angular packages into a single PR
      angular:
        patterns: ["@angular/*", "@angular-devkit/*"]
      # Group all development tools
      dev-tools:
        dependency-type: "development"
    labels:
      - "dependencies"
      - "automated"
    # Limit open PRs at once (prevents PR flood)
    open-pull-requests-limit: 10
```

Dependabot opens PRs automatically when new versions are available. Your CI runs tests on each PR. If all tests pass, a reviewer can merge with confidence. This keeps your dependency tree fresh without manual effort.

**Renovate** (by Mend) is an alternative to Dependabot with more sophisticated configuration — it can group related packages, schedule updates at specific times, use automerge for patch updates, and handle monorepos more elegantly:

```json
// renovate.json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:base"],
  "packageRules": [
    {
      "matchPackagePatterns": ["^@angular/"],
      "groupName": "Angular packages",
      "groupSlug": "angular"
    },
    {
      "matchUpdateTypes": ["patch"],
      "automerge": true  // auto-merge patch updates if CI passes
    }
  ],
  "schedule": ["before 9am on Monday"]
}
```

---

### Security Audits: npm audit, Snyk, Dependabot

A security vulnerability in any of your hundreds of dependencies is a vulnerability in your application. Auditing your dependency tree for known vulnerabilities is not optional in professional development.

**`npm audit` — the built-in scanner:**

```bash
# Run a security audit
npm audit

# Example output:
# found 3 vulnerabilities (1 low, 1 moderate, 1 critical)
#
# critical  Prototype pollution in lodash
# Package: lodash
# Patched in: >=4.17.21
# Dependency of: my-app (direct)
# Fix available: npm audit fix
#
# Run `npm audit fix` to fix them

# Apply automatic fixes (updates package-lock.json)
npm audit fix

# Show full details in JSON (for CI integration)
npm audit --json

# Only show high and critical vulnerabilities (fail CI on these)
npm audit --audit-level=high
```

**Integrating npm audit into CI:**

```yaml
# .github/workflows/security.yml
- name: Security audit
  run: npm audit --audit-level=high
  # Exits non-zero if any high/critical vulnerabilities found
  # → CI fails, PR cannot be merged until fixed
```

**Snyk — deeper vulnerability scanning:**

Snyk provides more detailed vulnerability information, fix suggestions, and IDE integration:

```bash
# Install Snyk CLI
npm install -g snyk

# Authenticate
snyk auth

# Test for vulnerabilities
snyk test

# Monitor your project continuously (sends results to Snyk dashboard)
snyk monitor

# Fix vulnerabilities automatically
snyk fix
```

**Vulnerability severity levels:**

| Level | CVSS Score | Meaning | Response |
|---|---|---|---|
| Critical | 9.0–10.0 | Remote code execution, data breach risk | Fix immediately, block deployment |
| High | 7.0–8.9 | Significant impact likely | Fix within days |
| Moderate | 4.0–6.9 | Limited impact or difficult to exploit | Fix within weeks |
| Low | 0.1–3.9 | Minimal impact | Fix at next update cycle |

---

### License Compliance

Every npm package has a license. Using a package in violation of its license can expose your organisation to legal risk. In commercial products, certain licenses are problematic:

| License Type | Examples | Can use in commercial product? |
|---|---|---|
| MIT | Most npm packages | ✅ Yes — permissive, just keep attribution |
| ISC | Various utilities | ✅ Yes — essentially equivalent to MIT |
| Apache 2.0 | Many Google packages | ✅ Yes — permissive with patent grant |
| BSD 2/3-Clause | Various | ✅ Yes — permissive |
| LGPL | Some C libraries | ⚠️ Check — dynamic linking usually OK |
| GPL v2/v3 | Some packages | ❌ Requires your code to be GPL too (copyleft) |
| AGPL | Some SaaS tools | ❌ Network use triggers copyleft |
| SSPL | MongoDB | ❌ Service use triggers copyleft |
| Unlicensed / "All rights reserved" | Rare | ❌ Cannot legally use without permission |

**Automating license checking:**

```bash
# Install license-checker
npm install -g license-checker

# Print all licenses in your dependency tree
license-checker --summary

# Check for any GPL licenses (fail if found)
license-checker --failOn "GPL"

# Generate a full license report (useful for legal team)
license-checker --csv > licenses.csv
```

```json
// package.json — integrate into CI
{
  "scripts": {
    "check-licenses": "license-checker --failOn 'GPL;AGPL' --excludePrivatePackages"
  }
}
```

---

### Bundle Size and Dependency Cost Awareness

Every npm package you add increases your JavaScript bundle size, which directly impacts page load time, LCP, and Time to Interactive. The question "can I install this package?" should always be accompanied by "how much does it cost?"

**Measuring package cost before installing:**

**bundlephobia.com** is the go-to tool — enter any npm package name and see its minified size, gzipped size, install time, and a breakdown of its own dependencies' sizes.

```bash
# Local bundle analysis
npm run build -- --stats-json          # Angular CLI generates webpack stats
npx webpack-bundle-analyzer dist/stats.json  # Interactive treemap visualisation

# Or use source-map-explorer
npm run build
npx source-map-explorer dist/browser/main-*.js
```

**Bundle size impact examples:**

| Package | Min + Gzip | Notes |
|---|---|---|
| `lodash` (full) | 25.2kB | 🔴 Never import this way in Angular |
| `lodash-es` (tree-shaken) | ~1kB per function | ✅ Use lodash-es and named imports |
| `moment.js` | 66.4kB | 🔴 Avoid — use date-fns or Temporal instead |
| `date-fns` (tree-shaken) | ~1-3kB per function | ✅ Tree-shakeable |
| `rxjs` (tree-shaken in Angular) | ~10kB | ✅ Angular already includes this |
| `zone.js` | 13kB | 🟡 Remove with zoneless Angular |

**Rules for responsible dependency usage:**

```typescript
// BAD — imports entire lodash library (25kB+)
import _ from 'lodash';
const sorted = _.sortBy(users, 'name');

// GOOD — tree-shaken single function (~2kB)
import { sortBy } from 'lodash-es';
const sorted = sortBy(users, 'name');

// BEST — use native JavaScript (0kB)
const sorted = [...users].sort((a, b) => a.name.localeCompare(b.name));
```

**Angular bundle budget configuration:**

Configure build budgets in `angular.json` to enforce maximum bundle sizes:

```json
// angular.json — budget configuration
{
  "configurations": {
    "production": {
      "budgets": [
        {
          "type": "initial",
          "maximumWarning": "1mb",
          "maximumError": "2mb"
        },
        {
          "type": "anyComponentStyle",
          "maximumWarning": "2kb",
          "maximumError": "4kb"
        },
        {
          "type": "anyScript",
          "maximumWarning": "100kb",
          "maximumError": "150kb"
        }
      ]
    }
  }
}
```

When the build exceeds these limits, it prints a warning (yellow) or fails entirely (red). This makes bundle size regressions immediately visible.

---

### Avoiding Dependency Bloat

Dependency bloat is the gradual accumulation of packages that are no longer needed, imperfectly sized, or duplicated. It slows builds, inflates bundles, and increases the attack surface for security vulnerabilities.

**Finding and removing unused dependencies:**

```bash
# depcheck — find unused dependencies and missing declarations
npx depcheck

# Output:
# Unused dependencies:
#   * moment
#   * @types/uuid (uuid itself is in node_modules but never imported)
# Missing dependencies:
#   * date-fns (imported in code but not in package.json)
```

**Finding duplicate packages (multiple versions of the same library):**

Duplicate packages are common in large dependency trees — package A requires lodash@3, package B requires lodash@4, and both end up in your bundle. Tools like `npm dedupe` can help:

```bash
# Attempt to flatten duplicate packages in node_modules
npm dedupe

# Check for duplicates with a visual tool
npx find-duplicate-dependencies
```

**The "Is this package worth its cost?" checklist:**

Before adding any new dependency, ask:

1. **Can I implement this myself in <30 lines?** If yes, do it — you avoid a dependency entirely.
2. **Is the package actively maintained?** Check last publish date, open issues, and GitHub stars. An unmaintained package is a security liability.
3. **Does it support tree shaking (ES Modules export)?** Check its `package.json` for `"module"` or `"exports"` fields with `.mjs` files.
4. **What is the bundle size?** Check bundlephobia.com before installing.
5. **Does it have its own heavy dependencies?** A small utility that pulls in a huge library is a bad trade.
6. **Is there an Angular-native way to do this?** Before reaching for an npm package, check if Angular CDK, Angular Material, or Angular's built-in APIs already provide the functionality.

> **Best Practice:** Periodically (quarterly is a good cadence) run `npx depcheck` and `npm outdated` together. Remove unused packages, update what you can, and consciously decide which major upgrades to schedule. Treat your `package.json` like code — it needs review and maintenance, not just accumulation.

---

## Summary & Key Takeaways

Understanding package managers and module systems bridges the gap between "I write Angular components" and "I understand the entire toolchain my application depends on." When a CI build fails because `package-lock.json` is out of sync, when a bundle is 3MB because someone imported all of lodash, when a security vulnerability is found in a transitive dependency — these are the moments where this knowledge becomes immediately practical.

### Quick Reference: npm Command Cheatsheet

```bash
# Installation
npm install               # install from package-lock.json
npm ci                    # clean install (CI/CD — never updates lockfile)
npm install lodash        # add to dependencies
npm install -D vitest     # add to devDependencies
npm uninstall lodash      # remove package

# Information
npm outdated              # list packages with newer versions
npm audit                 # check for security vulnerabilities
npm list --depth=0        # show installed packages (top level only)
npm ls lodash             # where does lodash come from in the dep tree?

# Execution
npm run build             # run a script
npm start                 # shortcut for npm run start
npm test                  # shortcut for npm run test
npx ng generate component # run binary without installing globally

# Publishing
npm login                 # authenticate
npm pack --dry-run        # preview what would be published
npm publish               # publish to registry
npm version patch         # bump version: 1.0.0 → 1.0.1
npm version minor         # bump version: 1.0.0 → 1.1.0
npm version major         # bump version: 1.0.0 → 2.0.0
```

### Module System Decision Matrix

| Situation | Module System to Use |
|---|---|
| Angular application code | ES Modules (TypeScript `import`/`export`) |
| Angular library publishing | ES Modules (`.mjs`, APF format) |
| Node.js config files (webpack, jest) | CommonJS (`require`/`module.exports`) or `.mjs` |
| Angular CLI configuration (`angular.json`) | JSON (not a module — just config) |
| TypeScript config files in Node.js context | CommonJS or type=module ESM |
| Old library you are maintaining | Likely CommonJS or UMD — modernise to ESM when feasible |
| CDN browser bundle with no bundler | UMD or IIFE (self-contained) |

### The Golden Rules

| Rule | Why It Matters |
|---|---|
| Always commit `package-lock.json` / `yarn.lock` / `pnpm-lock.yaml` | Reproducible, deterministic installs across all environments |
| Always use `npm ci` in CI/CD, not `npm install` | Faster, reliable, never silently updates the lockfile |
| Use `^` for most deps, `~` for infrastructure (rxjs, TypeScript) | Balance staying current vs stability |
| Check bundlephobia before adding any new package | Prevents silent bundle size regressions |
| Run `npm audit --audit-level=high` in CI | Catches critical security issues before they reach production |
| Prefer tree-shakeable ES Module packages | Dead code elimination requires static ES Module structure |
| Use `npx` instead of global installs for CLI tools | Version consistency across team, no machine pollution |
| Never put auth tokens directly in `.npmrc` | Use environment variable interpolation `${NPM_TOKEN}` |

---

*Phase 8 Complete — Next: Phase 9 — Build Tools & Bundlers*

*Document version: March 2026 | Node.js 22 · npm 10 · pnpm 9 · Angular 21*
