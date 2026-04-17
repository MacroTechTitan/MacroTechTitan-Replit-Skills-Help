# SKILL: Fix Replit Deployment Issues

  ## Architecture — pnpm monorepo on Replit

  ```
  workspace/
    artifacts/
      optima-quant/          # React SPA  — port 22211
        .replit-artifact/artifact.toml
        server.mjs           # static file server (production)
        vite.config.ts       # requires PORT + BASE_PATH env vars
        dist/public/         # built output (created by vite build)
      api-server/            # Express API — port 8080
        .replit-artifact/artifact.toml
        dist/index.mjs       # built output (created by tsc)
    .replit                  # top-level config (workflows + deployment)
  ```

  ## How Replit routes traffic in production
  - `/api/*`  → port 8080  (Express)
  - `/*`      → port 22211 (server.mjs static SPA)
  These are defined in each artifact's `artifact.toml` → `[[services]]` → `paths`.

  ## Critical rules

  ### NEVER use npm in a pnpm workspace
  ```bash
  # WRONG — breaks catalog: dependencies
  cd artifacts/optima-quant && npm install && npm run build

  # CORRECT
  PORT=22211 BASE_PATH=/ NODE_ENV=production \\
    pnpm --filter @workspace/optima-quant run build
  ```

  ### Vite build requires env vars
  vite.config.ts throws if PORT or BASE_PATH are missing.
  Always set both when building manually:
  ```bash
  PORT=22211 BASE_PATH=/ NODE_ENV=production \\
    pnpm --filter @workspace/optima-quant run build
  ```

  ### DO NOT add static file serving to Express
  The Express API (port 8080) must NOT serve the React SPA.
  server.mjs on port 22211 already handles all SPA traffic.
  The Replit router sends /api/* to Express and /* to server.mjs.

  ### DO NOT set NODE_ENV=production globally
  vite.config.ts checks NODE_ENV !== 'production' to load dev plugins.
  Setting it globally kills local dev.
  It is already set correctly per-process in artifact.toml:
  ```toml
  [services.production.run.env]
  NODE_ENV = "production"
  ```

  ## Production deployment pipeline (already configured)
  From artifacts/optima-quant/.replit-artifact/artifact.toml:
  ```toml
  [services.production.build]
  args = ["pnpm", "--filter", "@workspace/optima-quant", "run", "build"]

  [services.production.run]
  args = ["node", "artifacts/optima-quant/server.mjs"]

  [services.production.run.env]
  PORT = "22211"
  NODE_ENV = "production"
  BASE_PATH = "/"
  ```
  Replit runs the build step automatically on every publish.

  ## Git situation on Replit
  - Remote is gitsafe-backup (internal) — NOT GitHub
  - git fetch origin FAILS — there is no "origin" remote
  - git reset --hard is BLOCKED in main agent context
  - rm -f .git/index.lock is BLOCKED in main agent context
  - To pull from GitHub: use the GitHub REST API (see github-sync skill)

  ## Environment variables — where they live
  | Var | Location | Notes |
  |---|---|---|
  | NODE_ENV | artifact.toml per-process | Do NOT set globally |
  | PORT | artifact.toml per-process | 22211 (frontend), 8080 (API) |
  | BASE_PATH | artifact.toml dev + prod | "/" for optima-quant |
  | APP_URL | .replit [userenv.shared] | https://optimaquant.com |
  | DATABASE_URL | Replit DB (auto-injected) | |
  | STRIPE_SECRET_KEY | Replit Secrets | Needed for Stripe sync |

  ## Verify production is healthy
  ```bash
  # 1. Check site is live
  curl -s https://optimaquant.com/ | grep -i "title"

  # 2. Check SPA routing (any client-side route returns 200)
  curl -s -o /dev/null -w "%{http_code}" https://optimaquant.com/changelog

  # 3. Check API healthz
  curl -s https://optimaquant.com/api/healthz
  ```

  ## Smoke-test server.mjs locally
  ```bash
  PORT=22220 node artifacts/optima-quant/server.mjs &
  sleep 1
  curl -s -o /dev/null -w "HTTP %{http_code}" http://localhost:22220/
  kill %1
  ```

  ## Stripe production connection error
  Non-fatal — API continues without Stripe sync.
  Fix: configure Stripe production credentials in the deployment secrets via the Stripe integration panel.

  ## Deployment logs
  Use fetch_deployment_logs tool (Replit agent) to see production logs.
  Key patterns to watch:
  - all artifact ports detected = healthy startup
  - Server listening port=8080 = API up
  - Optima Quant frontend server listening on port 22211 = SPA up
  