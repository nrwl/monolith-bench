# Nx React Repository

<a alt="Nx logo" href="https://nx.dev" target="_blank" rel="noreferrer"><img src="https://raw.githubusercontent.com/nrwl/nx/master/images/nx-logo.png" width="45"></a>

✨ A repository showcasing key [Nx](https://nx.dev) features for React monorepos ✨

## 📊 Benchmark baseline

This repository is the **baseline** for the Nx caching and distributed task execution benchmark.
CI here runs one target per VM with no distribution, so each job's wall time measures the work
done on a single machine.

The optimized counterpart, which runs the same workload through Nx Cloud agents, lives at
[nrwl/cache-dte-bench-2](https://github.com/nrwl/cache-dte-bench-2). See that repository for
more information about the benchmark and its results.

**Monolith.** The workspace has a single project, `shop`, plus the root project that
holds the `validate` target. There are no libraries: all code lives in the app's source tree
(`apps/shop/src`), organized by folder and imported file-to-file with relative paths (no
barrel `index.ts` files), and the Playwright specs live in the same project (`apps/shop/e2e`).

Nx Cloud is disabled (`neverConnectToCloud` in `nx.json`, `NX_NO_CLOUD` in CI), so there is
no remote caching and no distribution.

## 📦 Project Overview

This repository demonstrates a production-ready React monorepo with:

- **1 Application**

  - `shop` - React e-commerce application with product listings and detail views. The
    product pages (`/products`) fetch from `http://localhost:3333/api`, which this workspace
    does not include, so they render an error state. The benchmarked `/features/*` pages need
    no backend.

- **Handwritten code** in `apps/shop/src`:

  - `features/products`, `features/product-detail` - Product listing and detail pages
  - `hooks` - Data-fetching hooks (`use-products`, `use-product`)
  - `components/shared` - Shared UI components (product card/grid, spinner, error message)
  - `models` - Shared data models
  - `test-utils` - Shared testing utilities

- **E2E Testing**
  - `apps/shop/e2e` - Playwright tests, run by the `e2e` / `e2e-ci` targets of `shop`

- **Generated code** (~300k lines of TypeScript in `apps/shop/src`)

  - `features/<domain>/<kind>/` - 300 features (30 domains x 10 kinds), each mounted at `/features/<domain>-<kind>`
  - `components/<group>/<kind>/` - 130 UI components (13 groups x 10 kinds)
  - `utils/<group>/` - 70 utilities (7 groups x 10 kinds)
  - `apps/shop/e2e/features/*.spec.ts` - 100 Playwright specs sampled evenly across the features (1 test each)

  Every e2e spec imports the route and item count of the feature it exercises straight from its source files.
  Each generated e2e test spends `E2E_TEST_DURATION_MS` (default 25000ms) waiting and
  `E2E_TEST_CPU_SECONDS` (default 2s) computing, interleaved across `E2E_TEST_BLOCKS`
  (default 10) rounds, see `apps/shop/e2e/support/pacing.ts`. That is ~27s per test,
  of which ~7% scales with hardware, matching real Playwright being mostly wait-bound.
  The 100 tests take roughly 45 minutes on one Playwright worker in CI. Set
  `E2E_TEST_DURATION_MS=0 E2E_TEST_CPU_SECONDS=0` to run at full speed locally.

### Simulated load

Unit tests model a realistic Vitest profile rather than pure sleep. Each spec file costs:

| Phase               | Env var                 | Default | Scales with hardware |
| ------------------- | ----------------------- | ------- | -------------------- |
| Fixed-work CPU burn | `UNIT_TEST_CPU_SECONDS` | 5s      | yes                  |
| Idle wait           | `UNIT_TEST_SLEEP_MS`    | 7300ms  | no                   |

The two are interleaved across `UNIT_TEST_BLOCKS` (default 10) rounds of compute-then-sleep,
which is closer to how a real Vitest run alternates between CPU work and waiting.

There are 1,111 spec files at 12.3s each. They all belong to the `shop` project,
so all of that time falls into the single `shop:test` task.

The CPU half runs a fixed number of iterations (`tools/test-delay/burn.mjs`), not a
fixed duration, so faster runners finish it sooner. That is what makes runner
comparisons meaningful; a `setTimeout` would take the same wall time on any machine.
The loop is a serial dependent integer chain with no allocation, so it tracks clock
speed and multiply latency rather than allocator, memory bandwidth and GC.

`UNITS_PER_SECOND` in `burn.mjs` is calibrated for one GitHub Actions vCPU. To
re-calibrate, run this **on a runner** and paste in the result:

```bash
node tools/test-delay/calibrate.mjs
```

By default Vitest runs one spec file at a time, the same as `cache-dte-bench`, so the burn
genuinely gets one core. Set `UNIT_TEST_WORKERS=<n>` to run `n` spec files at once, e.g.
`UNIT_TEST_WORKERS=4 npx nx test shop`. Set `UNIT_TEST_CPU_SECONDS=0
UNIT_TEST_SLEEP_MS=0` to run unit tests at full speed locally.
The generated features, components, utilities, app routes and e2e specs are produced by `tools/generate-shop.mjs`. Regenerate with:

```bash
node tools/generate-shop.mjs
```

## 🚀 Quick Start

```bash
# Clone the repository
git clone <your-fork-url>
cd <your-repository-name>

# Install dependencies
npm install

# Serve the React shop application
npx nx run shop:serve

# Build all projects
npx nx run-many -t build

# Run tests
npx nx run-many -t test

# Lint all projects
npx nx run-many -t lint

# Run e2e tests
npx nx run shop:e2e

# Run tasks in parallel

npx nx run-many -t lint test build e2e --parallel=3

# Visualize the project graph
npx nx graph
```

## ⭐ Featured Nx Capabilities

This repository showcases several powerful Nx features:

### 1. 🎭 Playwright E2E Testing

End-to-end testing with Playwright is pre-configured:

```bash
# Run e2e tests
npx nx run shop:e2e

# Run e2e tests in CI mode
npx nx run shop:e2e-ci
```

[Learn more about E2E testing →](https://nx.dev/docs/technologies/test-tools/playwright)

### 2. ⚡ Vitest for Unit Testing

Fast unit testing with Vitest:

```bash
# Test the shop app (every spec under apps/shop/src)
npx nx run shop:test

# Test all projects
npx nx run-many -t test
```

[Learn more about Vite testing →](https://nx.dev/docs/technologies/build-tools/vite)

## 📁 Project Structure

```
├── apps/
│   └── shop/                  - React e-commerce app
│       ├── e2e/               - Playwright specs
│       └── src/
│           ├── app/                    - App shell and routes
│           ├── features/               - Feature pages (products, product-detail, 300 generated)
│           ├── components/             - UI components (shared + 130 generated)
│           ├── utils/                  - 70 generated utilities
│           ├── hooks/                  - Data-fetching hooks
│           ├── models/                 - Shared models
│           └── test-utils/             - Testing utilities
├── tools/                     - Generator and simulated-load helpers
├── nx.json                    - Nx configuration
├── tsconfig.json              - TypeScript configuration
└── eslint.config.mjs          - ESLint configuration
```

## 📚 Useful Commands

```bash
# Project exploration
npx nx graph                                    # Interactive dependency graph
npx nx list                                     # List installed plugins
npx nx show project shop --web                 # View project details

# Development
npx nx run shop:serve                              # Serve React app
npx nx run shop:build                              # Build React app
npx nx run shop:test                               # Test the shop app
npx nx run shop:lint                               # Lint the shop app

# Running multiple tasks
npx nx run-many -t build                       # Build all projects
npx nx run-many -t test --parallel=3          # Test in parallel
npx nx run-many -t lint test build            # Run multiple targets

# Affected commands (great for CI)
npx nx affected -t build                       # Build only affected projects
npx nx affected -t test                        # Test only affected projects
```

## 🎯 Adding New Features

### Generate a new React application:

```bash
npx nx g @nx/react:app my-app
```

### Generate a new React component:

```bash
npx nx g @nx/react:component apps/shop/src/components/my-component
```

You can use `npx nx list` to see all available plugins and `npx nx list <plugin-name>` to see all generators for a specific plugin.

## Install Nx Console

Nx Console is an editor extension that enriches your developer experience. It lets you run tasks, generate code, and improves code autocompletion in your IDE. It is available for VSCode and IntelliJ.

[Install Nx Console &raquo;](https://nx.dev/docs/getting-started/editor-setup?utm_source=nx_project&utm_medium=readme&utm_campaign=nx_projects)

## 🔗 Learn More

- [Nx Documentation](https://nx.dev/docs)
- [Crafting Your Workspace Tutorial](https://nx.dev/docs/getting-started/tutorials/crafting-your-workspace)
- [Module Boundaries](https://nx.dev/docs/features/enforce-module-boundaries)
- [Playwright Testing](https://nx.dev/docs/technologies/test-tools/playwright)
- [Vite](https://nx.dev/docs/technologies/build-tools/vite)
- [Docker Integration](https://nx.dev/docs/guides/nx-release/release-docker-images)

## 💬 Community

Join the Nx community:

- [Discord](https://go.nx.dev/community)
- [X (Twitter)](https://twitter.com/nxdevtools)
- [LinkedIn](https://www.linkedin.com/company/nrwl)
- [YouTube](https://www.youtube.com/@nxdevtools)
- [Blog](https://nx.dev/blog)
