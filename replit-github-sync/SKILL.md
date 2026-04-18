---
name: github-sync
description: Pull latest changes from GitHub (MacroTechTitan/optima-quant master) and rebuild the Optima Quant frontend. Use when the user asks to pull from GitHub, sync latest code, rebuild the dashboard, or says the dashboard is showing old code.
---

# GitHub Sync — Optima Quant

## Critical Environment Facts

- **No `origin` remote** is configured in this Replit environment. All GitHub sync MUST go through the GitHub REST API.
- **Sandbox blocks destructive git commands**: `rm -f .git/index.lock`, `git reset --hard`, `git rebase`, `git restore`, etc. will be rejected by the sandbox.
- **Stale lock files** (`.git/index.lock`) cause `git status` to fail with a sandbox protection error — work around this by using the REST API approach instead of native git pull.
- **This is a pnpm workspace** — never run `npm install` inside an artifact subdirectory. Use `pnpm` from the workspace root.
- **Frontend runs as a Vite dev server (HMR)** — there is no static `dist/` build in development. `npm run build` requires a `PORT` env var and is only for production deploys. Just restart the workflow after writing files.
- **Repo**: `MacroTechTitan/optima-quant`, branch: `master`
- **GitHub connection**: use `listConnections('github')` in `code_execution` to get the `access_token`.

## Step-by-Step Workflow

### Step 1 — Check current local state
```bash
git log --oneline -5
git remote -v
```
Note the local HEAD SHA. If `git status` fails due to index.lock, proceed to REST API approach.

### Step 2 — Get GitHub master HEAD via REST API
Run in `code_execution`:
```javascript
const conns = await listConnections('github');
const token = conns[0].settings.access_token;
const repo = 'MacroTechTitan/optima-quant';

const resp = await fetch(`https://api.github.com/repos/${repo}/git/refs/heads/master`, {
  headers: { Authorization: `Bearer ${token}`, 'User-Agent': 'optima-quant-agent' }
});
const ref = await resp.json();
console.log('GitHub master SHA:', ref.object?.sha);

const commitResp = await fetch(`https://api.github.com/repos/${repo}/commits/${ref.object?.sha}`, {
  headers: { Authorization: `Bearer ${token}`, 'User-Agent': 'optima-quant-agent' }
});
const commit = await commitResp.json();
console.log('Message:', commit.commit?.message);
console.log('Date:', commit.commit?.author?.date);
```

### Step 3 — Find changed files between local HEAD and GitHub HEAD
Run in `code_execution`:
```javascript
// Get files changed in the latest commit(s)
const remoteSha = '<from step 2>';
const resp = await fetch(
  `https://api.github.com/repos/${repo}/commits/${remoteSha}`,
  { headers: { Authorization: `Bearer ${token}`, 'User-Agent': 'optima-quant-agent' } }
);
const commit = await resp.json();
console.log('Changed files:');
(commit.files || []).forEach(f => console.log(` [${f.status}] ${f.filename}`));
```
If there are multiple new commits, repeat for each parent SHA.

### Step 4 — Fetch and write changed files
For each changed/added file, fetch content from GitHub and write locally:
```javascript
async function getFile(path) {
  const resp = await fetch(
    `https://api.github.com/repos/${repo}/contents/${path}?ref=master`,
    { headers: { Authorization: `Bearer ${token}`, 'User-Agent': 'optima-quant-agent' } }
  );
  const data = await resp.json();
  return Buffer.from(data.content, 'base64').toString('utf-8');
}
const content = await getFile('artifacts/optima-quant/src/App.tsx');
console.log(content); // then use write() tool to write it
```
Store content in `globalThis` to pass between code_execution blocks.

**Skip these files when pulling**: `.replit`, `replit.nix`, `.env`, anything in `client/` (if present).

### Step 5 — Fix missing imports in App.tsx (if updated)
After writing App.tsx, check routes vs imports. Common gaps that GitHub may have:
- `SimulatorPage from "@/pages/simulator"`
- `ScannerPage from "@/pages/scanner"`
- `ApiClientsPage from "@/pages/api-clients"`

Always check `src/pages/` locally (`ls artifacts/optima-quant/src/pages/`) and ensure every page referenced in routes is imported.

### Step 6 — Verify new pages exist locally; pull from GitHub if missing
```bash
ls artifacts/optima-quant/src/pages/
ls artifacts/optima-quant/src/components/ | grep task
```
If pages referenced in App.tsx don't exist locally, fetch them from GitHub (step 4 pattern).

### Step 7 — Restart the frontend workflow (NOT build)
```
restart_workflow("artifacts/optima-quant: web")
```
HMR picks up all file changes. No build step needed in development.

### Step 8 — Confirm pages in browser
Take screenshots of:
- `/changelog` (public, no auth)
- `/pricing` (public, no auth)
- `/tycoon`, `/metatrader`, `/data` (redirect to login — confirm no 404)

## Adding `origin` Remote (optional, if git pull is needed)

If the sandbox git operations become available, you can add origin:
```python
import os, subprocess
pat = os.environ.get('GH_PASSWORD', '')
subprocess.run(['git', 'remote', 'add', 'origin',
    f'https://MacroTechTitan:{pat}@github.com/MacroTechTitan/optima-quant.git'])
```
Then `git fetch origin` (read-only, allowed). **Do NOT use `git reset --hard`** — it's blocked. Instead write files manually via the REST API approach above.

## Pushing Changes to GitHub

Use the REST API blob→tree→commit→PATCH pattern (see existing push scripts or `code_execution` history). Never use `git push` directly.

## Common Issues & Fixes

| Symptom | Cause | Fix |
|---|---|---|
| `git status` fails with sandbox error | Stale `.git/index.lock` | Use REST API approach; don't try to rm the lock |
| Routes return 404 on prod | Router not mounted in `routes/index.ts` | Add `router.use(newRouter)` and restart API workflow |
| `npm install` fails with `catalog:` error | Wrong package manager in pnpm workspace | Use `pnpm --filter @workspace/optima-quant install` |
| `npm run build` fails with PORT error | Vite requires PORT env var for build | Don't build in dev; just restart the dev server workflow |
| New pages show login instead of content | Expected — authenticated routes require login | Confirm no 404; 302→login is correct |
| App.tsx has undefined component errors | Missing imports for pages used in routes | Add the import lines manually before writing the file |
