# Fix Replit — Quick Reference

  ## Most common mistakes

  | Mistake | Correct |
  |---|---|
  | npm install in artifact dir | pnpm --filter @workspace/<name> install |
  | npm run build | PORT=22211 BASE_PATH=/ NODE_ENV=production pnpm --filter @workspace/optima-quant run build |
  | Add static serving to Express | DON'T — server.mjs handles this on port 22211 |
  | Set NODE_ENV=production globally | DON'T — breaks Vite dev plugins |
  | git fetch origin | FAILS — no origin remote on Replit |
  | git reset --hard | BLOCKED — use GitHub REST API instead |
  | rm -f .git/index.lock | BLOCKED in main agent context |

  ## Build the frontend
  ```bash
  PORT=22211 BASE_PATH=/ NODE_ENV=production \\
    pnpm --filter @workspace/optima-quant run build
  ```

  ## Build the API
  ```bash
  pnpm --filter @workspace/api-server run build
  ```

  ## Verify dist was built
  ```bash
  ls -lh artifacts/optima-quant/dist/public/
  ```

  ## Check production is live
  ```bash
  curl -s https://optimaquant.com/ | grep title
  curl -s -o /dev/null -w "%{http_code}" https://optimaquant.com/changelog
  ```

  ## Port map
  | Service | Port |
  |---|---|
  | Frontend (server.mjs in prod, Vite in dev) | 22211 |
  | Express API | 8080 |
  | ClaudeCoon Flask pipeline | 9000 |
  | Mockup sandbox | 8081 |

  ## Artifact config location
  - Frontend: artifacts/optima-quant/.replit-artifact/artifact.toml
  - API: artifacts/api-server/.replit-artifact/artifact.toml

  ## Workflows
  - Project → parallel → triggers ClaudeCoon
  - ClaudeCoon → cd nt-strategy-pipeline && python replit/replit_runner.py
  - artifacts/optima-quant: web → pnpm --filter @workspace/optima-quant run dev
  - artifacts/api-server: API Server → pnpm --filter @workspace/api-server run dev
  