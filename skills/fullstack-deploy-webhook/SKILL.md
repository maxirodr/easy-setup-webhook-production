---
name: easy-setup-webhook-production
description: Add or update a deploy webhook endpoint for Laravel or Node.js (Express + PM2) backend + frontend projects. Use when the user asks to "add deploy webhook", "deploy endpoint", "auto deploy", "setup PM2 deploy", "webhook para deployar", "agregar deploy webhook", "update deploy webhook", "upgrade deploy webhook", "actualizar deploy webhook", "actualizar webhook", or similar. Implements git pull + conditional builds with security (throttle, ban, token validation) and battle-tested gotchas (safe.directory, PATH, permissions, PM2 reload).
metadata:
  author: maxirodr
  version: "1.2.0"
---

# Easy Setup Webhook Production v1.2 — Multi-Stack Wizard

Skill to add a secure, production-ready deploy webhook endpoint to any **Laravel** or **Node.js (Express + PM2)** backend with one or more frontends. The endpoint does a git pull and conditionally runs builds based on which files actually changed.

**Two modes:**
- **New installation** (Phases 1–4): Full interactive wizard. Use when there's no deploy webhook in the project.
- **Update existing installation** (Phase 5): Upgrades a v1.0/v1.1 installation to v1.2. Use when the user says "update", "upgrade", or "actualizar" the deploy webhook.

---

## Phase 1: Spec Mode (Interactive Wizard)

**CRITICAL: Do NOT generate any code until the entire spec is confirmed by the user.** Collect all configuration first, show a summary, get approval, then generate.

### Step 1.1 — Detect the Backend Stack

Scan the project to determine the backend framework:

1. **Check for Laravel**: Look for `composer.json` containing `laravel/framework` in `require`.
2. **Check for Node.js/Express**: Look for `package.json` (at root or common subdirs) containing `express` in `dependencies`.
3. **If both or neither detected**: Ask the user which stack to use.
4. **Detect git remote**: Run `git remote get-url origin` to extract user/repo (e.g., `user/repo`).
5. **Detect git branch**: Run `git branch --show-current` for the default deploy branch.

Store the detected stack as `spec.backend` = `"laravel"` or `"nodejs"`.

### Step 1.2 — Backend-Specific Questions

#### If Laravel:

Ask the user (showing detected defaults):

1. **Structure**: Is Laravel at the git root, or in a subdirectory? (Detect by checking if `artisan` is at root or in a subdir like `backend/`.)
   - If subdir: record `spec.laravel.subdir` (e.g., `"backend"`)
   - If root: `spec.laravel.subdir` = `null`
2. **Controller namespace**: Check existing controllers for the namespace pattern (e.g., `App\Http\Controllers\Api\V1`). Confirm with user.
3. **Git branch for deploy**: Default to detected branch or `main`. Confirm.

#### If Node.js (Express + PM2):

Ask the user (showing detected defaults):

1. **Entry file**: Detect by checking `package.json` `main` field, or look for `app.js`, `server.js`, `src/index.js`, `src/index.ts`. Confirm.
2. **PM2 process name**: Suggest based on `package.json` `name` field. Confirm.
3. **Port**: Check for `PORT` in `.env` or `.env.example`, or hardcoded in entry file. Confirm.
4. **TypeScript?**: Detect `tsconfig.json`. If yes, record `spec.nodejs.typescript = true`.
5. **Package manager**: Detect by lockfile (same logic as frontends — `package-lock.json` → npm, `yarn.lock` → yarn, `pnpm-lock.yaml` → pnpm, `bun.lockb` → bun). If no lockfile, ask. Store as `spec.nodejs.pkg_manager`.
6. **Route registration pattern**: Read the entry file to understand how routes are registered (e.g., `app.use('/api', router)` or `app.get(...)`). Ask where to inject the deploy route (e.g., `app.use('/api/deploy', deployRouter)`).
7. **Git branch for deploy**: Default to detected branch or `main`. Confirm.

### Step 1.3 — Frontend Configuration (ALWAYS ask)

**Always execute this step regardless of backend stack.**

1. **Scan for frontends**: Look for subdirectories containing `package.json` (exclude `node_modules`, the root if Node.js backend, and vendor dirs).
2. **If no frontends detected**: Ask "Does this project have any frontend apps that need building?" If no, set `spec.frontends = []` and skip the rest of this step. The `{FRONTEND_BLOCKS}` placeholder will be replaced with an empty comment in both templates.
3. **For each detected frontend**, ask ALL of these:
   - **Directory**: Confirm the detected path (e.g., `frontend/`, `landing/`, `admin/`)
   - **Package manager**: Detect by lockfile:
     - `package-lock.json` → `npm`
     - `yarn.lock` → `yarn`
     - `pnpm-lock.yaml` → `pnpm`
     - `bun.lockb` → `bun`
     - No lockfile → **ask the user**
   - **Build command**: Read `scripts.build` from the frontend's `package.json`, show it to the user, and confirm. Example: "I see `build: 'tsc && vite build'` — should I use `npm run build`?"
   - **Output directory**: Detect from framework config or common defaults (`dist/`, `build/`, `.next/`, `out/`). Confirm.
4. **Ask if there are additional frontends** not detected.

Store as `spec.frontends[]` with shape: `{ dir, pkg_manager, build_cmd, output_dir }`.

### Step 1.4 — Server Environment & URL Detection

Ask:

1. **Web server**: nginx, apache, or none (Node.js can serve directly). Confirm.
2. **Server user** (for SSH access): Who do they SSH as? (root, deploy user, etc.)
3. **Laravel public URL path** (Laravel only — CRITICAL):
   - **Do NOT assume** the URL structure. Different nginx configs serve Laravel at different paths.
   - Ask the user: "What is the base URL of your Laravel API? For example, if your health check is at `https://example.com/api/health`, the base URL is `https://example.com`."
   - Alternatively, ask them to run: `grep -A5 "try_files.*index.php" /etc/nginx/sites-enabled/*` to detect the nginx location block that serves Laravel.
   - Common patterns:
     - `location /` → `https://domain.com/api/deploy`
     - `location /apilaravel/public` → `https://domain.com/apilaravel/public/api/deploy`
     - `location /api` → `https://domain.com/api/deploy` (with different prefix config)
   - Store as `spec.laravel.public_url_base`
4. **PHP-FPM user** (Laravel only — CRITICAL):
   - **IMPORTANT**: For Laravel, the "server user" (SSH user) is NOT the user that runs the deploy. PHP-FPM runs as its own user (typically `www-data` on Debian/Ubuntu, `nginx` on CentOS/RHEL).
   - Detect by asking the user to run: `ps aux | grep php-fpm | grep -v root | head -1 | awk '{print $1}'`
   - Or check PHP-FPM pool config: `grep -r "^user" /etc/php/*/fpm/pool.d/`
   - **Explain to the user**: "PHP-FPM (which executes your Laravel code) runs as a specific system user, usually `www-data`. This user needs permission to run `git pull` and write to the project directory. This is different from the user you SSH with."
   - Store as `spec.laravel.fpm_user`
5. **Node.js process user** (Node.js only): What user runs PM2? Suggest current user or dedicated deploy user.
6. **Node.js/npm path on the server** (if frontends exist — CRITICAL):
   - Ask the user to run on the server: `which npm`
   - If the path contains `.nvm` (e.g., `/root/.nvm/versions/node/v20.19.5/bin/npm`), record:
     - `spec.nvm_bin_path` = the directory (e.g., `/root/.nvm/versions/node/v20.19.5/bin`)
     - `spec.nvm_home_dir` = the home dir containing `.nvm` (e.g., `/root`)
   - If npm is in `/usr/bin/` or `/usr/local/bin/`, no special handling needed.
   - **Explain to the user**: "I need to know where npm is installed on your server because PHP-FPM doesn't have the same PATH as your shell. If npm is installed via nvm, I'll need to hardcode the path."

### Step 1.5 — Confirm Spec Before Generating

Display a structured summary of the ENTIRE spec:

```
╔══════════════════════════════════════════════════╗
║            DEPLOY WEBHOOK SPEC SUMMARY           ║
╠══════════════════════════════════════════════════╣
║ Backend:     {laravel|nodejs}                    ║
║ Git Remote:  {user/repo}                         ║
║ Git Branch:  {branch}                            ║
╠══════════════════════════════════════════════════╣
║ BACKEND CONFIG                                   ║
║ (Laravel-specific or Node.js-specific details)   ║
╠══════════════════════════════════════════════════╣
║ FRONTENDS                                        ║
║ 1. {dir} — {pkg_manager} — {build_cmd}          ║
║ 2. ...                                           ║
╠══════════════════════════════════════════════════╣
║ SERVER                                           ║
║ User: {user}    Web Server: {web_server}         ║
╚══════════════════════════════════════════════════╝
```

**Ask for explicit confirmation before proceeding to code generation.**

---

## Phase 2A: Code Generation — Laravel Branch

Only execute this phase if `spec.backend === "laravel"`.

### Step 2A.1 — Create DeployController

Create `app/Http/Controllers/{spec.laravel.namespace}/DeployController.php`.

Resolve these placeholders from the spec:
- `{NAMESPACE}` → e.g., `App\Http\Controllers\Api\V1`
- `{BASE_PATH}` → `realpath(base_path())` if Laravel at root, `realpath(base_path('..'))` if in subdir
- `{GIT_BRANCH}` → from spec (e.g., `main`)
- `{BACKEND_PREFIX}` → `''` if Laravel at root, `'backend/'` if in subdir (for git diff prefix matching)
- `{BACKEND_CWD}` → `$basePath` if at root, `$basePath . '/backend'` if in subdir

#### Template

```php
<?php

namespace {NAMESPACE};

use App\Http\Controllers\Controller;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\Log;
use Symfony\Component\Process\Process;

class DeployController extends Controller
{
    /**
     * Execute deployment: git pull + conditional builds based on changed files.
     *
     * GET /api/deploy?token=xxx&git_token=xxx
     */
    public function deploy(Request $request): JsonResponse
    {
        $ip = $request->ip();

        // Check ban
        if (Cache::has("deploy_banned:{$ip}")) {
            return response()->json(['error' => 'Banned'], 403);
        }

        // Throttle: max 5 attempts per hour
        $throttleKey = "deploy_attempts:{$ip}";
        $attempts = (int) Cache::get($throttleKey, 0);

        if ($attempts >= 5) {
            Cache::put("deploy_banned:{$ip}", true, now()->addDays(7));
            Log::critical("Deploy: IP {$ip} banned for 7 days (exceeded throttle)");
            return response()->json(['error' => 'Banned'], 403);
        }

        Cache::put($throttleKey, $attempts + 1, now()->addHour());

        // Validate token
        $token = $request->query('token');

        if (!$token || $token !== config('app.deploy_token')) {
            Log::warning('Deploy: invalid token', ['ip' => $ip]);
            return response()->json(['error' => 'Unauthorized'], 401);
        }

        // Validate git_token (GitHub PAT passed via URL)
        $gitToken = $request->query('git_token');
        if (!$gitToken) {
            return response()->json(['error' => 'git_token required'], 400);
        }

        // Auth OK — reset throttle
        Cache::forget($throttleKey);

        // Prevent concurrent deploys
        $lock = Cache::lock('deploy_running', 600); // 10 min max
        if (!$lock->get()) {
            return response()->json(['error' => 'Deploy already in progress'], 409);
        }

        try {

        $basePath = {BASE_PATH};
        $results = [];
        $success = true;

        Log::info('Deploy: starting from ' . $ip);

        $gitSafe = ['git', '-c', "safe.directory={$basePath}"];

        // 1. Capture current HEAD before pull
        $headBefore = $this->runCommand([...$gitSafe, 'rev-parse', 'HEAD'], $basePath);

        // 2. Git pull with GitHub credentials
        $gitUser = config('app.deploy_git_user');
        $gitRepo = config('app.deploy_git_repo');
        $authUrl = "https://{$gitUser}:{$gitToken}@github.com/{$gitRepo}.git";

        $results['git_pull'] = $this->runCommand([...$gitSafe, 'pull', $authUrl, '{GIT_BRANCH}'], $basePath);
        if (!$results['git_pull']['success']) {
            return response()->json([
                'status' => 'failed',
                'error' => 'git pull failed',
                'results' => $results,
            ], 500);
        }

        // 3. Get changed files between old HEAD and new HEAD
        $headAfter = $this->runCommand([...$gitSafe, 'rev-parse', 'HEAD'], $basePath);
        $changedFiles = [];

        if ($headBefore['output'] !== $headAfter['output']) {
            $diff = $this->runCommand(
                [...$gitSafe, 'diff', '--name-only', $headBefore['output'], $headAfter['output']],
                $basePath
            );
            $changedFiles = array_filter(explode("\n", $diff['output']));
        }

        $results['changed_files'] = $changedFiles;

        if (empty($changedFiles)) {
            return response()->json([
                'status' => 'no_changes',
                'timestamp' => now()->toIso8601String(),
                'results' => $results,
            ]);
        }

        // Helper: check if any changed file matches a prefix
        $hasChangesIn = fn(string $prefix) => collect($changedFiles)
            ->contains(fn($f) => str_starts_with($f, $prefix));

        // 4. Composer update (only if composer files changed)
        if ($hasChangesIn('{BACKEND_PREFIX}composer')) {
            $results['composer_update'] = $this->runCommand(
                ['composer', 'update', '--no-dev', '--optimize-autoloader', '--no-interaction'],
                {BACKEND_CWD}
            );
        } else {
            $results['composer_update'] = ['skipped' => true, 'reason' => 'no changes in composer.*'];
        }

        // 5. Migrations (only if migration files changed)
        if ($hasChangesIn('{BACKEND_PREFIX}database/migrations')) {
            $results['artisan_migrate'] = $this->runCommand(
                ['php', 'artisan', 'migrate', '--force'],
                {BACKEND_CWD}
            );
            if (!$results['artisan_migrate']['success']) {
                $success = false;
            }
        } else {
            $results['artisan_migrate'] = ['skipped' => true, 'reason' => 'no new migrations'];
        }

        // 6. Clear Laravel cache (always — fast and safe)
        $results['artisan_optimize_clear'] = $this->runCommand(
            ['php', 'artisan', 'optimize:clear'],
            {BACKEND_CWD}
        );

        // 7+ Frontend builds (conditional per frontend directory)
        {FRONTEND_BLOCKS}

        $status = $success ? 'completed' : 'completed_with_errors';
        Log::info("Deploy: {$status}", ['results' => array_map(
            fn($r) => $r['skipped'] ?? ($r['success'] ?? null),
            $results
        )]);

        return response()->json([
            'status' => $status,
            'timestamp' => now()->toIso8601String(),
            'results' => $results,
        ], $success ? 200 : 500);

        } finally {
            $lock->release();
        }
    }

    /**
     * Run a shell command and return the result.
     */
    private function runCommand(array $command, string $cwd, int $timeout = 120): array
    {
        try {
            $env = getenv();
            // Add nvm (if applicable) + node_modules/.bin to PATH so npm scripts find tsc, vite, etc.
            // {NVM_PATH_LINE} — only include if nvm detected during wizard (e.g., $nvmBin = '/root/.nvm/versions/node/vXX/bin';)
            $env['PATH'] = $cwd . '/node_modules/.bin:' . ($env['PATH'] ?? '/usr/local/bin:/usr/bin:/bin');
            // Set npm cache + HOME to /tmp so www-data can write without permission issues
            $env['npm_config_cache'] = '/tmp/.npm-deploy-cache';
            $env['HOME'] = '/tmp';

            $process = new Process($command, $cwd, $env);
            $process->setTimeout($timeout);
            $process->run();

            return [
                'success' => $process->isSuccessful(),
                'output' => trim($process->getOutput()),
                'error' => trim($process->getErrorOutput()),
                'exit_code' => $process->getExitCode(),
            ];
        } catch (\Exception $e) {
            return [
                'success' => false,
                'output' => '',
                'error' => $e->getMessage(),
                'exit_code' => -1,
            ];
        }
    }
}
```

### Laravel Frontend Block Template

Replace `{FRONTEND_BLOCKS}` with one block per frontend from the spec. Each block:

```php
// {N}. {FrontendName} (only if changes in {dir}/)
${varName}Path = $basePath . '/{dir}';
if ($hasChangesIn('{dir}/') && is_dir(${varName}Path)) {
    $results['{key}_install'] = $this->runCommand(
        {INSTALL_CMD_ARRAY},
        ${varName}Path
    );
    {AUDIT_FIX_BLOCK}
    $results['{key}_build'] = $this->runCommand(
        ['{pkg_manager}', 'run', 'build'],
        ${varName}Path,
        300
    );
    if (!$results['{key}_build']['success']) {
        $success = false;
    }
} else {
    $results['{key}'] = ['skipped' => true, 'reason' => 'no changes in {dir}/'];
}
```

Resolve `{INSTALL_CMD_ARRAY}` per package manager — each argument MUST be a separate array element for Symfony Process:
- **npm**: `['npm', 'ci']`
- **yarn**: `['yarn', 'install', '--frozen-lockfile']`
- **pnpm**: `['pnpm', 'install', '--frozen-lockfile']`
- **bun**: `['bun', 'install', '--frozen-lockfile']`

Resolve `{AUDIT_FIX_BLOCK}` per package manager — only npm and pnpm support `audit fix`. The `npm install` only runs if `audit fix` actually modified `package.json` or the lockfile (detected via `md5_file`):

**For npm:**
```php
    // Check for vulnerabilities and fix if found
    $auditResult = $this->runCommand(['npm', 'audit'], ${varName}Path);
    if (!$auditResult['success']) {
        $pkgHash = md5_file(${varName}Path . '/package.json');
        $lockFile = ${varName}Path . '/package-lock.json';
        $lockHash = file_exists($lockFile) ? md5_file($lockFile) : null;
        $results['{key}_audit_fix'] = $this->runCommand(['npm', 'audit', 'fix'], ${varName}Path);
        if (md5_file(${varName}Path . '/package.json') !== $pkgHash
            || (file_exists($lockFile) ? md5_file($lockFile) : null) !== $lockHash) {
            $results['{key}_reinstall'] = $this->runCommand(['npm', 'install'], ${varName}Path);
        }
    }
```

**For pnpm:**
```php
    // Check for vulnerabilities and fix if found
    $auditResult = $this->runCommand(['pnpm', 'audit'], ${varName}Path);
    if (!$auditResult['success']) {
        $pkgHash = md5_file(${varName}Path . '/package.json');
        $lockFile = ${varName}Path . '/pnpm-lock.yaml';
        $lockHash = file_exists($lockFile) ? md5_file($lockFile) : null;
        $results['{key}_audit_fix'] = $this->runCommand(['pnpm', 'audit', 'fix'], ${varName}Path);
        if (md5_file(${varName}Path . '/package.json') !== $pkgHash
            || (file_exists($lockFile) ? md5_file($lockFile) : null) !== $lockHash) {
            $results['{key}_reinstall'] = $this->runCommand(['pnpm', 'install'], ${varName}Path);
        }
    }
```

**For yarn/bun:** Replace `{AUDIT_FIX_BLOCK}` with an empty comment:
```php
    // (No audit fix — not supported by {pkg_manager})
```

### Step 2A.2 — Add Route

In `routes/api.php`, add (outside any auth middleware group):

```php
Route::get('/deploy', [\App\Http\Controllers\{NAMESPACE_SUFFIX}\DeployController::class, 'deploy']);
```

### Step 2A.3 — Add Config Values

In `config/app.php`, add inside the config array:

```php
'deploy_token' => env('DEPLOY_TOKEN'),
'deploy_git_user' => env('DEPLOY_GIT_USER', '{detected_git_user}'),
'deploy_git_repo' => env('DEPLOY_GIT_REPO', '{detected_git_repo}'),
```

### Step 2A.4 — Update .env.example

Append:

```
DEPLOY_TOKEN=
DEPLOY_GIT_USER={detected_git_user}
DEPLOY_GIT_REPO={detected_git_repo}
```

---

## Phase 2B: Code Generation — Node.js (Express + PM2) Branch

Only execute this phase if `spec.backend === "nodejs"`.

### Step 2B.1 — Create Deploy Route Module

Create `routes/deploy.js` (or `routes/deploy.ts` if TypeScript project).

**IMPORTANT**: Use `execFile` (not `exec`) for all shell commands — `execFile` does not spawn a shell, preventing shell injection attacks.

#### Template (JavaScript)

```js
const { execFile } = require('child_process');
const { promisify } = require('util');
const { Router } = require('express');
const path = require('path');
const fs = require('fs');

const execFileAsync = promisify(execFile);
const router = Router();

const SECURITY_FILE = path.join(__dirname, '..', 'deploy-security.json');
const MAX_ATTEMPTS = 5;
const BAN_DURATION_MS = 7 * 24 * 60 * 60 * 1000; // 7 days
let deployInProgress = false;

/**
 * Load security state (bans + throttle) from persistent JSON file.
 */
function loadSecurity() {
    try {
        return JSON.parse(fs.readFileSync(SECURITY_FILE, 'utf8'));
    } catch {
        return { bans: {}, attempts: {} };
    }
}

function saveSecurity(data) {
    fs.writeFileSync(SECURITY_FILE, JSON.stringify(data, null, 2));
}

/**
 * Run a command safely using execFile (no shell).
 */
async function runCommand(file, args, cwd, timeoutMs = 120000) {
    try {
        const env = {
            ...process.env,
            PATH: `${cwd}/node_modules/.bin:${process.env.PATH || '/usr/local/bin:/usr/bin:/bin'}`,
        };
        const { stdout, stderr } = await execFileAsync(file, args, {
            cwd,
            env,
            timeout: timeoutMs,
        });
        return { success: true, output: stdout.trim(), error: stderr.trim() };
    } catch (err) {
        return {
            success: false,
            output: err.stdout?.trim() || '',
            error: err.stderr?.trim() || err.message,
            exit_code: err.code,
        };
    }
}

/**
 * GET /deploy?token=xxx&git_token=xxx
 */
router.get('/', async (req, res) => {
    const ip = req.ip;
    const security = loadSecurity();
    const now = Date.now();

    // Check ban
    if (security.bans[ip] && now < security.bans[ip]) {
        return res.status(403).json({ error: 'Banned' });
    }
    // Clean expired ban
    if (security.bans[ip] && now >= security.bans[ip]) {
        delete security.bans[ip];
    }

    // Throttle: max 5 attempts per hour
    const attempts = (security.attempts[ip] || []).filter(t => now - t < 3600000);
    if (attempts.length >= MAX_ATTEMPTS) {
        security.bans[ip] = now + BAN_DURATION_MS;
        saveSecurity(security);
        console.error(`Deploy: IP ${ip} banned for 7 days (exceeded throttle)`);
        return res.status(403).json({ error: 'Banned' });
    }

    security.attempts[ip] = [...attempts, now];
    saveSecurity(security);

    // Validate token
    const token = req.query.token;
    if (!token || token !== process.env.DEPLOY_TOKEN) {
        console.warn(`Deploy: invalid token from ${ip}`);
        return res.status(401).json({ error: 'Unauthorized' });
    }

    // Validate git_token
    const gitToken = req.query.git_token;
    if (!gitToken) {
        return res.status(400).json({ error: 'git_token required' });
    }

    // Auth OK — reset throttle for this IP
    security.attempts[ip] = [];
    saveSecurity(security);

    // Prevent concurrent deploys
    if (deployInProgress) {
        return res.status(409).json({ error: 'Deploy already in progress' });
    }
    deployInProgress = true;

    try {

    const basePath = fs.realpathSync(path.join(__dirname, '..'));
    const results = {};
    let success = true;

    console.log(`Deploy: starting from ${ip}`);

    const gitSafe = ['-c', `safe.directory=${basePath}`];

    // 1. Capture current HEAD before pull
    const headBefore = await runCommand('git', [...gitSafe, 'rev-parse', 'HEAD'], basePath);

    // 2. Git pull with GitHub credentials
    const gitUser = process.env.DEPLOY_GIT_USER;
    const gitRepo = process.env.DEPLOY_GIT_REPO;
    const authUrl = `https://${gitUser}:${gitToken}@github.com/${gitRepo}.git`;

    results.git_pull = await runCommand('git', [...gitSafe, 'pull', authUrl, '{GIT_BRANCH}'], basePath);
    if (!results.git_pull.success) {
        return res.status(500).json({ status: 'failed', error: 'git pull failed', results });
    }

    // 3. Get changed files between old HEAD and new HEAD
    const headAfter = await runCommand('git', [...gitSafe, 'rev-parse', 'HEAD'], basePath);
    let changedFiles = [];

    if (headBefore.output !== headAfter.output) {
        const diff = await runCommand(
            'git',
            [...gitSafe, 'diff', '--name-only', headBefore.output, headAfter.output],
            basePath
        );
        changedFiles = diff.output.split('\n').filter(Boolean);
    }

    results.changed_files = changedFiles;

    if (changedFiles.length === 0) {
        return res.json({ status: 'no_changes', timestamp: new Date().toISOString(), results });
    }

    // Helper: check if any changed file matches a prefix
    const hasChangesIn = (prefix) => changedFiles.some(f => f.startsWith(prefix));

    // 4. Backend: install deps if package.json or lockfile changed
    if (hasChangesIn('package') || hasChangesIn('{LOCK_FILE_PREFIX}')) {
        results.backend_install = await runCommand('{pkg_manager}', [{INSTALL_ARGS}], basePath);
        {BACKEND_AUDIT_FIX_BLOCK}
    } else {
        results.backend_install = { skipped: true, reason: 'no changes in package/lock files' };
    }

    // 5. TypeScript build (if applicable)
    {TS_BUILD_BLOCK}

    // 6+ Frontend builds (conditional per frontend directory)
    {FRONTEND_BLOCKS}

    // Final: PM2 reload (zero-downtime)
    results.pm2_reload = await runCommand('pm2', ['reload', '{PM2_PROCESS_NAME}'], basePath);
    if (!results.pm2_reload.success) {
        success = false;
    }

    const status = success ? 'completed' : 'completed_with_errors';
    console.log(`Deploy: ${status}`, JSON.stringify(
        Object.fromEntries(Object.entries(results).map(([k, v]) => [k, v.skipped ?? v.success ?? null]))
    ));

    res.status(success ? 200 : 500).json({ status, timestamp: new Date().toISOString(), results });

    } finally {
        deployInProgress = false;
    }
});

module.exports = router;
```

#### TypeScript Build Block

If `spec.nodejs.typescript === true`, replace `{TS_BUILD_BLOCK}` with:

```js
// TypeScript: compile if .ts files changed
if (changedFiles.some(f => f.endsWith('.ts'))) {
    results.tsc_build = await runCommand('npx', ['tsc'], basePath);
    if (!results.tsc_build.success) {
        // If TypeScript fails, do NOT reload PM2 — broken build
        return res.status(500).json({
            status: 'failed',
            error: 'TypeScript compilation failed',
            results,
        });
    }
} else {
    results.tsc_build = { skipped: true, reason: 'no .ts files changed' };
}
```

If `spec.nodejs.typescript === false`, replace `{TS_BUILD_BLOCK}` with an empty comment:

```js
// (No TypeScript build — plain JS project)
```

#### Backend Audit Fix Block

Resolve `{BACKEND_AUDIT_FIX_BLOCK}` per package manager — only npm and pnpm support `audit fix`. The reinstall only runs if `audit fix` actually modified `package.json` or the lockfile:

**For npm:**
```js
        // Check for vulnerabilities and fix if found
        const backendAudit = await runCommand('npm', ['audit'], basePath);
        if (!backendAudit.success) {
            const pkgBefore = fs.readFileSync(path.join(basePath, 'package.json'), 'utf8');
            const lockPath = path.join(basePath, 'package-lock.json');
            const lockBefore = fs.existsSync(lockPath) ? fs.readFileSync(lockPath, 'utf8') : '';
            results.backend_audit_fix = await runCommand('npm', ['audit', 'fix'], basePath);
            const pkgAfter = fs.readFileSync(path.join(basePath, 'package.json'), 'utf8');
            const lockAfter = fs.existsSync(lockPath) ? fs.readFileSync(lockPath, 'utf8') : '';
            if (pkgBefore !== pkgAfter || lockBefore !== lockAfter) {
                results.backend_reinstall = await runCommand('npm', ['install'], basePath);
            }
        }
```

**For pnpm:**
```js
        // Check for vulnerabilities and fix if found
        const backendAudit = await runCommand('pnpm', ['audit'], basePath);
        if (!backendAudit.success) {
            const pkgBefore = fs.readFileSync(path.join(basePath, 'package.json'), 'utf8');
            const lockPath = path.join(basePath, 'pnpm-lock.yaml');
            const lockBefore = fs.existsSync(lockPath) ? fs.readFileSync(lockPath, 'utf8') : '';
            results.backend_audit_fix = await runCommand('pnpm', ['audit', 'fix'], basePath);
            const pkgAfter = fs.readFileSync(path.join(basePath, 'package.json'), 'utf8');
            const lockAfter = fs.existsSync(lockPath) ? fs.readFileSync(lockPath, 'utf8') : '';
            if (pkgBefore !== pkgAfter || lockBefore !== lockAfter) {
                results.backend_reinstall = await runCommand('pnpm', ['install'], basePath);
            }
        }
```

**For yarn/bun:** Replace `{BACKEND_AUDIT_FIX_BLOCK}` with an empty comment:
```js
        // (No audit fix — not supported by {pkg_manager})
```

#### TypeScript Variant

If the project uses TypeScript, generate `routes/deploy.ts` instead with proper imports:

```ts
import { execFile } from 'child_process';
import { promisify } from 'util';
import { Router, Request, Response } from 'express';
import path from 'path';
import fs from 'fs';
```

And use `export default router;` instead of `module.exports`.

### Node.js Frontend Block Template

Replace `{FRONTEND_BLOCKS}` with one block per frontend from the spec:

```js
// {N}. {FrontendName} (only if changes in {dir}/)
const {varName}Path = path.join(basePath, '{dir}');
if (hasChangesIn('{dir}/') && fs.existsSync({varName}Path)) {
    results.{key}_install = await runCommand('{pkg_manager}', [{install_args}], {varName}Path);
    {AUDIT_FIX_BLOCK}
    results.{key}_build = await runCommand(
        '{pkg_manager}',
        ['run', 'build'],
        {varName}Path,
        300000
    );
    if (!results.{key}_build.success) {
        success = false;
    }
} else {
    results.{key} = { skipped: true, reason: 'no changes in {dir}/' };
}
```

Resolve `{AUDIT_FIX_BLOCK}` per package manager — only npm and pnpm support `audit fix`. The reinstall only runs if `audit fix` actually modified `package.json` or the lockfile:

**For npm:**
```js
    // Check for vulnerabilities and fix if found
    const {varName}Audit = await runCommand('npm', ['audit'], {varName}Path);
    if (!{varName}Audit.success) {
        const pkgBefore = fs.readFileSync(path.join({varName}Path, 'package.json'), 'utf8');
        const lockPath = path.join({varName}Path, 'package-lock.json');
        const lockBefore = fs.existsSync(lockPath) ? fs.readFileSync(lockPath, 'utf8') : '';
        results.{key}_audit_fix = await runCommand('npm', ['audit', 'fix'], {varName}Path);
        const pkgAfter = fs.readFileSync(path.join({varName}Path, 'package.json'), 'utf8');
        const lockAfter = fs.existsSync(lockPath) ? fs.readFileSync(lockPath, 'utf8') : '';
        if (pkgBefore !== pkgAfter || lockBefore !== lockAfter) {
            results.{key}_reinstall = await runCommand('npm', ['install'], {varName}Path);
        }
    }
```

**For pnpm:**
```js
    // Check for vulnerabilities and fix if found
    const {varName}Audit = await runCommand('pnpm', ['audit'], {varName}Path);
    if (!{varName}Audit.success) {
        const pkgBefore = fs.readFileSync(path.join({varName}Path, 'package.json'), 'utf8');
        const lockPath = path.join({varName}Path, 'pnpm-lock.yaml');
        const lockBefore = fs.existsSync(lockPath) ? fs.readFileSync(lockPath, 'utf8') : '';
        results.{key}_audit_fix = await runCommand('pnpm', ['audit', 'fix'], {varName}Path);
        const pkgAfter = fs.readFileSync(path.join({varName}Path, 'package.json'), 'utf8');
        const lockAfter = fs.existsSync(lockPath) ? fs.readFileSync(lockPath, 'utf8') : '';
        if (pkgBefore !== pkgAfter || lockBefore !== lockAfter) {
            results.{key}_reinstall = await runCommand('pnpm', ['install'], {varName}Path);
        }
    }
```

**For yarn/bun:** Replace `{AUDIT_FIX_BLOCK}` with an empty comment:
```js
    // (No audit fix — not supported by {pkg_manager})
```

### Step 2B.2 — Register Route in Express App

In the user's entry file (e.g., `app.js`), add:

```js
const deployRouter = require('./routes/deploy');
app.use('/api/deploy', deployRouter);
```

Place it based on `spec.nodejs.route_pattern` — near existing route registrations.

### Step 2B.3 — Add .env Entries

Append to `.env.example` (create if it doesn't exist):

```
DEPLOY_TOKEN=
DEPLOY_GIT_USER={detected_git_user}
DEPLOY_GIT_REPO={detected_git_repo}
PM2_PROCESS_NAME={spec.nodejs.pm2_name}
```

### Step 2B.4 — Create ecosystem.config.js (if not exists)

Only create if file doesn't already exist:

```js
module.exports = {
    apps: [
        {
            name: '{PM2_PROCESS_NAME}',
            script: '{ENTRY_FILE}',
            env: {
                NODE_ENV: 'production',
                PORT: {PORT},
            },
        },
    ],
};
```

---

## Phase 3: Frontend Build Blocks — Package Manager Reference

When generating frontend blocks in either Laravel or Node.js templates, use the correct commands per package manager:

| Package Manager | Install Command          | Run Build         | Audit Fix Support |
|-----------------|--------------------------|-------------------|-------------------|
| `npm`           | `npm ci`                 | `npm run build`   | Yes (`npm audit fix` → `npm install`) |
| `yarn`          | `yarn install --frozen-lockfile` | `yarn run build` | No |
| `pnpm`          | `pnpm install --frozen-lockfile` | `pnpm run build` | Yes (`pnpm audit fix` → `pnpm install`) |
| `bun`           | `bun install --frozen-lockfile`  | `bun run build`  | No |

**For Laravel (Symfony Process)**, the command array splits differently:
- `npm`: `['npm', 'ci']` / `['npm', 'run', 'build']`
- `yarn`: `['yarn', 'install', '--frozen-lockfile']` / `['yarn', 'run', 'build']`
- `pnpm`: `['pnpm', 'install', '--frozen-lockfile']` / `['pnpm', 'run', 'build']`
- `bun`: `['bun', 'install', '--frozen-lockfile']` / `['bun', 'run', 'build']`

**Audit fix commands (Laravel)** — only for npm/pnpm:
- `npm`: `['npm', 'audit']` → `['npm', 'audit', 'fix']` → `['npm', 'install']`
- `pnpm`: `['pnpm', 'audit']` → `['pnpm', 'audit', 'fix']` → `['pnpm', 'install']`

**For Node.js (execFile)**, the command splits as:
- `npm`: `runCommand('npm', ['ci'], ...)` / `runCommand('npm', ['run', 'build'], ...)`
- `yarn`: `runCommand('yarn', ['install', '--frozen-lockfile'], ...)` / `runCommand('yarn', ['run', 'build'], ...)`
- `pnpm`: `runCommand('pnpm', ['install', '--frozen-lockfile'], ...)` / `runCommand('pnpm', ['run', 'build'], ...)`
- `bun`: `runCommand('bun', ['install', '--frozen-lockfile'], ...)` / `runCommand('bun', ['run', 'build'], ...)`

**Audit fix commands (Node.js)** — only for npm/pnpm:
- `npm`: `runCommand('npm', ['audit'], ...)` → `runCommand('npm', ['audit', 'fix'], ...)` → `runCommand('npm', ['install'], ...)`
- `pnpm`: `runCommand('pnpm', ['audit'], ...)` → `runCommand('pnpm', ['audit', 'fix'], ...)` → `runCommand('pnpm', ['install'], ...)`

**Build timeout**: Always 300 seconds (5 minutes) for frontend builds. Small VPS instances can take 2-5 minutes for Next.js builds.

---

## Phase 4: Post-Implementation Instructions

After generating all code, provide setup instructions tailored to the detected stack.

### Shared Instructions (both stacks)

**Generate a deploy token:**

```bash
openssl rand -hex 32
```

**How to trigger a deploy:**

```
GET https://your-domain.com/api/deploy?token=YOUR_TOKEN&git_token=YOUR_GITHUB_PAT
```

The `git_token` is a GitHub Personal Access Token (classic) with `repo` scope. It's passed per-request (not stored in env) for extra security.

**Security notes:**
- IP-based throttle: 5 failed attempts per hour, then 7-day ban
- Successful auth resets the throttle counter
- The GitHub PAT is passed per-request, not stored

### Laravel-Specific Instructions — Complete Server Setup Checklist

**Present this as a numbered step-by-step guide to the user. ALL steps must be completed BEFORE testing the deploy endpoint.**

```
╔══════════════════════════════════════════════════════════╗
║       SERVER SETUP CHECKLIST (run all on the server)     ║
╠══════════════════════════════════════════════════════════╣
║ 1. Set .env variables                                    ║
║ 2. Give PHP-FPM user ownership of project                ║
║ 3. Fix nvm permissions (if applicable)                   ║
║ 4. Clear Laravel cache                                   ║
║ 5. Test the deploy URL                                   ║
╚══════════════════════════════════════════════════════════╝
```

**Step 1 — Set .env variables:**
```bash
cd /path/to/laravel
# Add these to .env (or edit with nano .env):
# DEPLOY_TOKEN=<output of: openssl rand -hex 32>
# DEPLOY_GIT_USER={detected_git_user}
# DEPLOY_GIT_REPO={detected_git_repo}
```

**Step 2 — Give PHP-FPM user ownership of project:**

**CRITICAL: File Permissions for PHP-FPM**

PHP-FPM runs as its own user (detected during setup as `{FPM_USER}`, typically `www-data`). This is NOT the same as the user you SSH with. The deploy webhook runs inside PHP-FPM, so `{FPM_USER}` needs to:
- Read/write the `.git` directory (for `git pull`)
- Write to project files (for builds, cache, logs)

```bash
# 1. Check which user PHP-FPM runs as (should match what we detected)
ps aux | grep php-fpm | grep -v root | head -1 | awk '{print $1}'

# 2. Give PHP-FPM user ownership of the project directory
#    ⚠️  This allows the web server process to read/write ALL project files.
#    This is REQUIRED for the deploy webhook to work (git pull, npm ci, builds).
#    If you're uncomfortable with this, consider a cron-based deploy approach instead.
sudo chown -R {FPM_USER}:{FPM_USER} /path/to/project

# 3. Set up .env on the server
cp .env.example .env
php artisan key:generate
# Edit .env: set DEPLOY_TOKEN, DEPLOY_GIT_USER, DEPLOY_GIT_REPO

# 4. Clear cache after first setup
php artisan optimize:clear
```

**Why `chown` is needed**: When you trigger the deploy webhook via HTTP, the code runs as `{FPM_USER}` (not as root or your SSH user). Git requires write access to `.git/` for `git pull`, and npm/composer need write access for installing dependencies. Without `chown`, you'll get "Permission denied" errors.

**Step 3 — Fix nvm permissions (ONLY if npm is installed via nvm):**

If `which npm` on the server returns a path containing `.nvm` (e.g., `/root/.nvm/versions/node/v20/bin/npm`), the PHP-FPM user needs permission to traverse the home directory and execute the binaries:

```bash
# Allow www-data to traverse /root/ without listing its contents
chmod 711 /root

# Allow www-data to execute npm/node binaries
chmod -R 755 /root/.nvm/versions/node/{VERSION}/bin/
```

Replace `/root` with the actual home directory if nvm is installed under a different user. Replace `{VERSION}` with the actual node version (e.g., `v20.19.5`).

**Why**: Even though we hardcode the nvm path in the deploy code, Linux still requires traverse permission (`x`) on every parent directory. `chmod 711` gives "execute" (traverse) without "read" (list), so www-data can reach `/root/.nvm/...` without seeing other files in `/root/`.

**Step 4 — Clear Laravel cache:**
```bash
cd /path/to/laravel
php artisan optimize:clear
```

**Step 5 — Test the deploy URL:**
```bash
# First test without git_token to verify the endpoint responds:
curl -s "{DEPLOY_URL}?token=YOUR_TOKEN" | python3 -m json.tool
# Expected: {"error":"git_token required"} — this means the endpoint works!

# Then test with both tokens:
curl -s "{DEPLOY_URL}?token=YOUR_TOKEN&git_token=YOUR_GITHUB_PAT" | python3 -m json.tool
# Expected: {"status":"no_changes",...} or {"status":"completed",...}
```

If you get a 404, check your nginx config — the URL path depends on how nginx routes to Laravel's `public/index.php`. See Gotcha #3b.

### Node.js-Specific Instructions

```bash
# Install PM2 globally if not already installed
npm install -g pm2

# Start the app with PM2
pm2 start ecosystem.config.js

# Configure PM2 to start on boot
pm2 startup
pm2 save

# Set file ownership (replace 'deploy' with your server user)
sudo chown -R {SERVER_USER}:{SERVER_USER} /path/to/project

# Add deploy-security.json to .gitignore
echo 'deploy-security.json' >> .gitignore
```

---

## Phase 5: Update Existing Installations (v1.1 → v1.2)

Use this phase when the user already has a deploy webhook installed and wants to upgrade. The update adds the **npm audit fix** feature (auto-fix vulnerabilities after install, before build).

**CRITICAL: Do NOT run the full wizard (Phases 1–4). Only modify the existing deploy file.**

### Step 5.1 — Find Existing Deploy File

**Laravel:**
1. Search for `DeployController.php` recursively in `app/Http/Controllers/`
2. Confirm it contains the deploy logic (look for `runCommand` method and `git_pull` key)

**Node.js:**
1. Check for `routes/deploy.js` or `routes/deploy.ts`
2. Confirm it contains the deploy logic (look for `runCommand` function and `git_pull` key)

If no deploy file is found, inform the user and suggest running the full wizard (Phases 1–4) instead.

### Step 5.2 — Check if Already Updated

Search the deploy file for `audit_fix` or `audit`. If found, the installation already has the v1.2 feature. Inform the user: "Your deploy webhook already has the audit fix feature. No update needed."

### Step 5.3 — Detect Frontend Blocks & Package Managers

Read the entire deploy file and identify all frontend blocks by looking for the install→build pattern.

#### Laravel — Pattern to find:

Each frontend block has this structure (variable names and keys vary per project):

```php
$results['XXXXX_install'] = $this->runCommand(
    ['PACKAGE_MANAGER', ...],
    $XXXXXPath
);
$results['XXXXX_build'] = $this->runCommand(
```

Extract from each block:
- **`$XXXXXPath`** — the variable name (e.g., `$frontendPath`, `$landingPath`, `$adminPath`)
- **`XXXXX`** — the result key prefix (e.g., `frontend`, `landing`, `admin`)
- **`PACKAGE_MANAGER`** — the package manager (e.g., `npm`, `yarn`, `pnpm`, `bun`)

#### Node.js — Patterns to find:

**Frontend blocks:**
```js
results.XXXXX_install = await runCommand('PACKAGE_MANAGER', [...], XXXXXPath);
results.XXXXX_build = await runCommand(
```

**Backend install block:**
```js
results.backend_install = await runCommand('PACKAGE_MANAGER', [...], basePath);
```

Extract the same info: variable name, result key prefix, package manager.

### Step 5.4 — Insert Audit Fix Blocks

For each frontend block found, insert the audit fix code **between** the `_install` and `_build` lines. For the Node.js backend, insert **after** `backend_install` (inside the same `if` block).

**Only insert for npm and pnpm.** For yarn/bun, insert a comment: `// (No audit fix — not supported by {pkg_manager})`

#### Laravel — Insert this block (for npm):

Replace `$VARPath` and `KEY` with the actual variable name and key prefix detected in Step 5.3:

```php
        // Check for vulnerabilities and fix if found
        $auditResult = $this->runCommand(['npm', 'audit'], $VARPath);
        if (!$auditResult['success']) {
            $pkgHash = md5_file($VARPath . '/package.json');
            $lockFile = $VARPath . '/package-lock.json';
            $lockHash = file_exists($lockFile) ? md5_file($lockFile) : null;
            $results['KEY_audit_fix'] = $this->runCommand(['npm', 'audit', 'fix'], $VARPath);
            if (md5_file($VARPath . '/package.json') !== $pkgHash
                || (file_exists($lockFile) ? md5_file($lockFile) : null) !== $lockHash) {
                $results['KEY_reinstall'] = $this->runCommand(['npm', 'install'], $VARPath);
            }
        }
```

For pnpm, use `pnpm` commands and `pnpm-lock.yaml` as the lockfile.

#### Node.js Frontend — Insert this block (for npm):

Replace `VAR` and `KEY` with the actual variable name and key prefix:

```js
    // Check for vulnerabilities and fix if found
    const VARAudit = await runCommand('npm', ['audit'], VARPath);
    if (!VARAudit.success) {
        const pkgBefore = fs.readFileSync(path.join(VARPath, 'package.json'), 'utf8');
        const lockPath = path.join(VARPath, 'package-lock.json');
        const lockBefore = fs.existsSync(lockPath) ? fs.readFileSync(lockPath, 'utf8') : '';
        results.KEY_audit_fix = await runCommand('npm', ['audit', 'fix'], VARPath);
        const pkgAfter = fs.readFileSync(path.join(VARPath, 'package.json'), 'utf8');
        const lockAfter = fs.existsSync(lockPath) ? fs.readFileSync(lockPath, 'utf8') : '';
        if (pkgBefore !== pkgAfter || lockBefore !== lockAfter) {
            results.KEY_reinstall = await runCommand('npm', ['install'], VARPath);
        }
    }
```

#### Node.js Backend — Insert after `results.backend_install` (for npm):

```js
        // Check for vulnerabilities and fix if found
        const backendAudit = await runCommand('npm', ['audit'], basePath);
        if (!backendAudit.success) {
            const pkgBefore = fs.readFileSync(path.join(basePath, 'package.json'), 'utf8');
            const lockPath = path.join(basePath, 'package-lock.json');
            const lockBefore = fs.existsSync(lockPath) ? fs.readFileSync(lockPath, 'utf8') : '';
            results.backend_audit_fix = await runCommand('npm', ['audit', 'fix'], basePath);
            const pkgAfter = fs.readFileSync(path.join(basePath, 'package.json'), 'utf8');
            const lockAfter = fs.existsSync(lockPath) ? fs.readFileSync(lockPath, 'utf8') : '';
            if (pkgBefore !== pkgAfter || lockBefore !== lockAfter) {
                results.backend_reinstall = await runCommand('npm', ['install'], basePath);
            }
        }
```

For pnpm, use `pnpm` commands and `pnpm-lock.yaml` as the lockfile.

### Step 5.5 — Show Summary

After making changes, show the user:

```
╔══════════════════════════════════════════════════╗
║       DEPLOY WEBHOOK UPDATED TO v1.2             ║
╠══════════════════════════════════════════════════╣
║ File modified: {path to deploy file}             ║
║ Audit fix blocks added: {count}                  ║
║ Package managers detected: {list}                ║
╠══════════════════════════════════════════════════╣
║ NEW BEHAVIOR                                     ║
║ After npm ci, runs npm audit to check for        ║
║ vulnerabilities. If found:                       ║
║   1. npm audit fix                               ║
║   2. npm install (only if files changed)         ║
║   3. npm run build (as before)                   ║
╠══════════════════════════════════════════════════╣
║ No server changes needed — deploy & test!        ║
╚══════════════════════════════════════════════════╝
```

No new env vars, permissions, or server changes are required. The update only adds code logic to the existing deploy file.

---

## Gotchas & Lessons Learned

These are hard-won lessons from production debugging. **Do not skip these.**

### Shared (both stacks)

#### 1. `safe.directory` with `realpath()`
Git refuses to operate in directories owned by a different user. The fix is `git -c safe.directory=X`, but the path **MUST** be the real absolute path. Use `realpath()` / `fs.realpathSync()` — not relative paths or paths with `..` segments, which git won't match against the ownership check.

#### 2. `node_modules/.bin` in PATH
Neither Symfony Process (Laravel) nor `execFile` (Node.js) inherits the shell's PATH modifications. npm scripts that call `tsc`, `vite`, `next`, etc. will fail with "command not found" unless you explicitly add `node_modules/.bin` to the PATH.

#### 3. File ownership on the server (PHP-FPM user ≠ SSH user)
The process that executes the deploy webhook needs write access to the project files. **This is NOT the user you SSH with.**
- **Laravel**: PHP-FPM runs as its own user (typically `www-data` on Debian/Ubuntu). Even if the user SSHs as `root`, the deploy runs as `www-data`. You MUST `chown -R www-data:www-data /path/to/project` or git will fail with "Permission denied" on `.git/FETCH_HEAD`.
  - Detect PHP-FPM user: `ps aux | grep php-fpm | grep -v root | head -1 | awk '{print $1}'`
  - Or: `grep -r "^user" /etc/php/*/fpm/pool.d/`
- **Node.js**: `sudo chown -R {pm2_user}:{pm2_user} /path/to/project`

**IMPORTANT for the wizard**: When asking "Server user", always clarify that for Laravel the relevant user is the PHP-FPM user, not the SSH user. The wizard MUST detect the PHP-FPM user and guide the user to set `chown` correctly BEFORE testing the deploy endpoint.

#### 3b. Detect the Laravel public URL path from nginx
Do NOT assume the URL structure. Always ask the user for the base URL of their API or detect it from the nginx config. Common patterns:
- `location /` with `try_files $uri $uri/ /index.php?$query_string` → API at `/api/deploy`
- `location /apilaravel/public` → API at `/apilaravel/public/api/deploy`
- Proxy pass to PHP-FPM on specific path → varies

The wizard should ask the user to confirm the final deploy URL before finishing.

#### 4. Node.js installed via nvm — not in PHP-FPM's PATH
If Node.js is installed via **nvm** (Node Version Manager), it lives in a user-specific path like `/root/.nvm/versions/node/vXX/bin/` or `/home/user/.nvm/...`. PHP-FPM's `www-data` user does NOT have this in its PATH, so `npm` will fail with "npm: not found".
- **Detection**: During the wizard, ask the user to run `which npm` on the server. If the path contains `.nvm`, you MUST hardcode that path into the `runCommand` method's PATH env.
- **Fix in code**: Add the nvm bin path to the process environment: `$env['PATH'] = $nvmBin . ':' . $env['PATH']`
- **Fix on server — REQUIRED permissions**: Even after adding the path, `www-data` needs permission to traverse the home directory and execute the binaries:
  ```bash
  # If nvm is in /root/.nvm/:
  chmod 711 /root                                          # Allow www-data to traverse /root/ (without listing contents)
  chmod -R 755 /root/.nvm/versions/node/vXX/bin/           # Allow www-data to execute npm/node
  ```
  Without `chmod 711 /root`, www-data gets "Permission denied" even though the path is correct. Without `chmod 755` on the bin dir, www-data can find npm but can't execute it.
- **Alternative**: Install Node.js system-wide via `apt install nodejs` or use the NodeSource PPA, which puts `npm` in `/usr/bin/` and avoids all these permission issues.
- **IMPORTANT**: The wizard MUST ask the user for the output of `which npm` on the server and include that path in the generated code. The post-implementation instructions MUST include the chmod commands if nvm is detected.

#### 5. npm cache permission for PHP-FPM / www-data
When npm runs as `www-data`, it tries to write its cache to `$HOME/.npm` (typically `/var/www/.npm`). This directory may not exist or may be owned by root. Instead of creating it, set the `npm_config_cache` and `HOME` env vars in the process to `/tmp`:
```php
$env['npm_config_cache'] = '/tmp/.npm-deploy-cache';
$env['HOME'] = '/tmp';
```
This is cleaner than creating `.npm` directories in `/var/www/`. The wizard MUST include these env vars in the `runCommand` method.

#### 6. Build timeout: 300 seconds
Frontend builds (especially Next.js) can take 2-5 minutes on small VPS instances. Always set a generous timeout. The default 60-second timeout WILL cause failures.

### Laravel-Specific

#### 7. Symfony Process `-c` flag: arguments must be separate array elements
```php
// CORRECT:
['git', '-c', "safe.directory={$basePath}", 'pull', ...]

// WRONG (will fail silently):
['git', '-c safe.directory=...', 'pull', ...]
['git', "-c safe.directory={$basePath}", 'pull', ...]
```

#### 8. No `npx` needed in npm scripts
If `package.json` has `"build": "tsc && vite build"`, running `npm run build` will find `tsc` in `node_modules/.bin` automatically. You do NOT need `npx tsc && npx vite build`. But the Symfony Process env still needs the PATH fix from gotcha #2.

#### 9. `--force` flag for migrations
In production, `php artisan migrate` requires the `--force` flag. Without it, the command hangs waiting for user input inside Symfony Process.

#### 10. `composer install` vs `composer update`
- `composer install` = **deterministic**, installs exact versions from `composer.lock`. Safe for automated deploys.
- `composer update` = resolves new dependency versions, may introduce untested code. **Never use in automated deploys.**
Always use `composer install --no-dev --optimize-autoloader` in deploy webhooks. This is the PHP equivalent of `npm ci` vs `npm install`.

### Node.js-Specific

#### 9. PM2 `reload` vs `restart`
- `pm2 reload` = **zero-downtime** — starts new process, routes traffic, then kills old
- `pm2 restart` = **downtime** — kills process first, then starts new
Always use `reload` in the deploy webhook.

#### 10. PM2 ecosystem file for env vars
PM2's `ecosystem.config.js` env vars take **precedence** over `.env` when using `dotenv`. If you define `PORT` in both places, the ecosystem file wins. Keep env vars in one place to avoid confusion.

#### 11. Server user ≠ www-data
Node.js apps typically don't run as `www-data`. Use a dedicated deploy user or the user that PM2 was started under. Running Node.js as `www-data` can cause issues with PM2's daemon.

#### 12. Build TypeScript before PM2 reload
If the project uses TypeScript, the build **must succeed** before reloading PM2. If `tsc` fails, do NOT reload — the old compiled JS is still running and functional. Reloading with broken compilation will crash the app.

#### 13. `npm ci` vs `npm install`
- `npm ci` = **deterministic**, deletes `node_modules` and installs from lock file. Faster and safer for CI/CD.
- `npm install` = resolves dependencies, may update lock file. Not suitable for automated deploys.
Always use `npm ci` (or equivalent frozen-lockfile flag) in deploy webhooks.

### Shared (both stacks)

#### 14. `npm audit fix` in deploy context
After `npm ci` (or equivalent install), the deploy runs `npm audit` to check for known vulnerabilities. If vulnerabilities are found (`npm audit` returns non-zero exit code), the deploy automatically runs:
1. `npm audit fix` — attempts to patch vulnerable dependencies
2. **Checks if `package.json` or the lockfile was actually modified** (via `md5_file` in PHP, content comparison in Node.js)
3. Only if files changed: `npm install` — reinstalls with the updated dependencies
4. Then proceeds to `npm run build` as normal

**Important caveats:**
- The fix modifies `package.json` and `package-lock.json` locally on the server, but these changes are **NOT committed**. The next `git pull` will overwrite them. This is intentional — it provides an immediate security patch while the developer pushes a proper fix.
- Only npm and pnpm support `audit fix`. For yarn and bun, the audit block is omitted.
- `npm audit` may report vulnerabilities that `audit fix` can't resolve (major version bumps required). In that case, `package.json`/lockfile won't change, so the reinstall is skipped — no wasted time.
- For composer (Laravel), dependency install already runs conditionally when `composer.json` or `composer.lock` change in the git pull (see step 4 of the Laravel template).
