# Vercel pnpm Monorepo Deployment Skill

## Overview
Deploy a pnpm workspace monorepo (frontend + API) to Vercel with a Python pipeline on Render.

## Architecture
- **Vercel**: React/Vite frontend (static) + Node.js API (serverless)
- **Render**: Python pipeline service only
- **Split**: Vercel handles web traffic; Render handles long-running background work

---

## vercel.json (repo root)

```json
{
  "version": 2,
  "installCommand": "pnpm install --no-frozen-lockfile",
  "buildCommand": "pnpm --filter @workspace/api-server run build && pnpm --filter @workspace/optima-quant run build",
  "outputDirectory": "artifacts/optima-quant/dist/public",
  "framework": null,
  "env": {
    "PORT": "3000",
    "BASE_PATH": "/",
    "NODE_ENV": "production"
  },
  "rewrites": [
    { "source": "/api/:path*", "destination": "/api/index" },
    { "source": "/((?!api).*)", "destination": "/index.html" }
  ]
}
```

### Key fields

| Field | Value | Why |
|---|---|---|
| `installCommand` | `pnpm install --no-frozen-lockfile` | Avoids lockfile conflicts on Vercel |
| `buildCommand` | build API then frontend | API must build first |
| `outputDirectory` | must match Vite `outDir` exactly | e.g. `artifacts/optima-quant/dist/public` |
| `framework` | `null` | Prevents Vercel auto-detecting wrong framework |
| `env` | PORT, BASE_PATH, NODE_ENV | Prevents vite.config.ts from throwing at build time |

---

## api/index.ts (repo root)

Vercel serverless function entry — re-exports the compiled Express app:

```ts
import app from '../artifacts/api-server/dist/index.mjs'
export default app
```

---

## vite.config.ts — must NOT throw on missing env vars

Build-time: PORT and BASE_PATH are not injected by Vercel. Use safe defaults:

```ts
// CORRECT
const rawPort = process.env.PORT ?? "22211";
const port = Number(rawPort);
const basePath = process.env.BASE_PATH ?? "/";

// WRONG — crashes Vercel build
if (!rawPort) { throw new Error("PORT is required"); }
```

---

## render.yaml — pipeline only (no Node service)

```yaml
services:
  - type: web
    name: claudecoon-pipeline
    runtime: python
    rootDir: nt-strategy-pipeline
    buildCommand: pip install -r requirements.txt
    startCommand: python replit/replit_runner.py
    autoDeploy: true

databases:
  - name: optimaquant-db
    databaseName: optimaquant
    user: optimaquant
    plan: free
```

---

## Common Errors

### "PORT environment variable is required"
Vite config throws if PORT is not set. Vercel does not inject PORT at build time.
**Fix**: Replace hard throws with `?? "22211"` default.

### outputDirectory not found
Vercel cannot find built files.
**Fix**: Must match `outDir` in vite.config.ts exactly. Check with:
```bash
grep -i outDir artifacts/optima-quant/vite.config.ts
```

### API routes 404
`api/index.ts` missing or not exporting the Express app.
**Fix**: Create `api/index.ts` at repo root that re-exports the compiled app.

### pnpm frozen lockfile error
```
ERR_PNPM_OUTDATED_LOCKFILE
```
**Fix**: Use `pnpm install --no-frozen-lockfile` in `installCommand`.

### otplib ESM error (@scure/base)
otplib 13+ depends on `@scure/base` which is ESM-only — breaks CJS bundles.
**Fix**: Replace with `speakeasy@2.0.0` (pure CJS).
```bash
pnpm --filter @workspace/api-server remove otplib
pnpm --filter @workspace/api-server add speakeasy
```
Update imports in routes and externals in build.mjs.

---

## Push vercel.json via GitHub API

```python
import os, requests, base64
pat = os.environ['GH_PASSWORD']
headers = {'Authorization': f'token {pat}', 'Content-Type': 'application/json'}
repo = 'MacroTechTitan/optima-quant'
content = open('vercel.json', 'rb').read()
resp = requests.get(f'https://api.github.com/repos/{repo}/contents/vercel.json', headers=headers)
sha = resp.json().get('sha', '') if resp.status_code == 200 else ''
data = {'message': 'fix: vercel.json', 'content': base64.b64encode(content).decode()}
if sha:
    data['sha'] = sha
resp = requests.put(f'https://api.github.com/repos/{repo}/contents/vercel.json', headers=headers, json=data)
print(resp.status_code)
```
