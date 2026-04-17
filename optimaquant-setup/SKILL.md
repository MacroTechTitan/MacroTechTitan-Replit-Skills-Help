# SKILL: OptimaQuant Project Context

  ## Project overview
  OptimaQuant (optimaquant.com) is a professional algorithmic trading platform ($99/month SaaS).
  Built on Replit with Express + React + Python pipeline.

  ## Repository
  GitHub: MacroTechTitan/optima-quant (private)

  ## Stack
  - Frontend: React 18 + Vite + shadcn/ui + Tailwind → artifacts/optima-quant/src/
  - API server: Express 5 + TypeScript → artifacts/api-server/src/
  - Pipeline: Python Flask → nt-strategy-pipeline/replit/replit_runner.py
  - Database: PostgreSQL via Drizzle ORM → shared/schema.ts

  ## Port map
  | Service | Port | Routes |
  |---|---|---|
  | React SPA (Vite dev / server.mjs prod) | 22211 | /* |
  | Express API | 8080 | /api/* |
  | ClaudeCoon Flask pipeline | 9000 | internal |
  | Mockup sandbox | 8081 | dev only |

  ## Key routes
  | Route | Description |
  |---|---|
  | / | Marketing homepage |
  | /dashboard | Main trading dashboard |
  | /strategies | Strategy management + ClaudeCoon |
  | /connections | Broker + ATI connections |
  | /tycoon | Tycoon AI dashboard |
  | /data | Market data + simulator |
  | /marketplace | Strategy marketplace |
  | /trader/:username | Public performance profile |
  | /admin | Admin panel (jgelet only) |
  | /changelog | Public changelog |
  | /pricing | Pricing page |
  | /metatrader | MetaTrader EA bridge |
  | /task-tracker | Admin task clipboard |

  ## Required Replit Secrets
  NODE_ENV, APP_URL, GH_PASSWORD, GITHUB_TOKEN,
  ANTHROPIC_API_KEY, STRIPE_SECRET_KEY,
  ADMIN_USERNAME, ADMIN_PASSWORD, ADMIN_EMAIL,
  TYCOON_API_KEY, NT_ATI_HOST, NT_ATI_PORT,
  ENCRYPTION_KEY, API_JWT_SECRET,
  VITE_AUTH0_DOMAIN, VITE_AUTH0_CLIENT_ID,
  VITE_AUTH0_AUDIENCE, AUTH0_DOMAIN,
  AUTH0_AUDIENCE, AUTH0_ISSUER_BASE_URL

  ## ClaudeCoon live trading
  Account: 1598925
  Instrument: MES JUN26 5-minute
  Entry window: 09:30-10:14 ET weekdays
  ATI tunnel: 7.tcp.ngrok.io:23018

  ## Tycoon integration
  Dashboard: optimaquant.com/tycoon
  Client library: nt-strategy-pipeline/tycoon/tycoon_client.py
  API routes: artifacts/api-server/src/routes/tycoon.ts

  ## Deployment rules (see fix-replit/SKILL.md for full details)
  - Build: PORT=22211 BASE_PATH=/ NODE_ENV=production pnpm --filter @workspace/optima-quant run build
  - Run prod: node artifacts/optima-quant/server.mjs
  - Publish triggers auto-build via artifact.toml [services.production.build]
  - NEVER use npm install in this pnpm workspace
  - NEVER add static serving to the Express API
  