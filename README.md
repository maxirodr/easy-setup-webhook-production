# fullstack-deploy-webhook

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that adds a secure deploy webhook endpoint to any **Laravel + frontend** project.

One command in Claude Code and you get a production-ready endpoint that:

- Pulls from GitHub
- Detects which files changed (`git diff`)
- Conditionally runs only what's needed: `composer update`, `migrate`, `npm build` per frontend
- Protects itself with IP throttle + 7-day ban + token validation

## Install

```
/plugin add maxirodr/fullstack-deploy-webhook
```

## Usage

In any Laravel project, tell Claude:

```
add a deploy webhook
```

Or use the slash command:

```
/fullstack-deploy-webhook
```

Claude will automatically:

1. Detect your Laravel app and any frontend directories
2. Create a `DeployController` with the full deploy logic
3. Add the route, config values, and `.env.example` entries
4. Give you the production setup instructions

## What It Creates

| File | Action |
|------|--------|
| `app/Http/Controllers/Api/V1/DeployController.php` | Created |
| `routes/api.php` | Modified (adds `/api/deploy` route) |
| `config/app.php` | Modified (adds `deploy_token`, `deploy_git_user`, `deploy_git_repo`) |
| `.env.example` | Modified (adds `DEPLOY_TOKEN`, `DEPLOY_GIT_USER`, `DEPLOY_GIT_REPO`) |

## Supported Project Structures

### Monorepo (Laravel in subdirectory)

```
my-project/
  backend/        <- Laravel app
  frontend/       <- React, Vue, etc.
  landing/        <- Next.js, Astro, etc.
```

### Standard (Laravel at root)

```
my-project/       <- Laravel app
  frontend/       <- React, Vue, etc.
```

The skill auto-detects the structure and adjusts paths accordingly.

## Features

### Smart Conditional Builds

The endpoint compares `HEAD` before and after `git pull`, then only runs steps for directories that actually changed:

| Change detected in | Action |
|---|---|
| `composer.json` / `composer.lock` | `composer update --no-dev` |
| `database/migrations/` | `php artisan migrate --force` |
| `frontend/` | `npm install` + `npm run build` |
| `landing/` | `npm install` + `npm run build` |
| Nothing changed | Returns `no_changes`, skips everything |

Laravel cache is always cleared (`optimize:clear`) since it's fast and safe.

### Security

- **Token validation** via `DEPLOY_TOKEN` env variable
- **GitHub PAT** passed per-request (not stored on server)
- **IP throttle**: 5 failed attempts per hour
- **IP ban**: 7 days after exceeding throttle
- **Logging**: All deploy activity logged via Laravel Log

### Production Gotchas (Already Handled)

These are issues we hit in production and solved in the template:

| Problem | Solution in the skill |
|---|---|
| Git "unsafe directory" error (www-data vs root) | `git -c safe.directory=X` with `realpath()` |
| Symfony Process `-c` flag silently fails | Splits `-c` and value as separate array elements |
| `tsc`/`vite`/`next` "command not found" | Adds `node_modules/.bin` to Process PATH |
| `artisan migrate` hangs in production | Uses `--force` flag |
| Frontend build times out | 300s timeout (default is 60s) |

## Production Setup

After Claude creates the endpoint, you need to configure your server:

### 1. Generate a deploy token

```bash
openssl rand -hex 32
```

### 2. Add to `.env` on your server

```env
DEPLOY_TOKEN=<the-token-you-generated>
DEPLOY_GIT_USER=your-github-username
DEPLOY_GIT_REPO=your-username/your-repo
```

### 3. Fix file ownership

```bash
sudo chown -R www-data:www-data /var/www/your-project
```

The web server (Apache/Nginx) runs as `www-data`. Without this, git and PHP will have permission issues.

### 4. Create a GitHub Personal Access Token

Go to [GitHub Settings > Tokens](https://github.com/settings/tokens) and create a **classic** token with `repo` scope.

### 5. Trigger a deploy

```
GET https://your-domain.com/api/deploy?token=YOUR_DEPLOY_TOKEN&git_token=YOUR_GITHUB_PAT
```

You can call this from:
- A browser bookmark
- A GitHub Actions workflow
- A cURL script
- Any HTTP client

### Example response

```json
{
  "status": "completed",
  "timestamp": "2025-01-15T10:30:00+00:00",
  "results": {
    "git_pull": { "success": true, "output": "Updating abc123..def456" },
    "changed_files": ["frontend/src/App.tsx", "frontend/package.json"],
    "composer_update": { "skipped": true, "reason": "no changes in composer.*" },
    "artisan_migrate": { "skipped": true, "reason": "no new migrations" },
    "artisan_optimize_clear": { "success": true },
    "frontend_install": { "success": true },
    "frontend_build": { "success": true }
  }
}
```

## Troubleshooting

### "Banned" (403)

Your IP exceeded 5 failed attempts. The ban lasts 7 days. To unban manually:

```bash
php artisan tinker
>>> Cache::forget('deploy_banned:YOUR_IP');
```

### "git pull failed"

Common causes:
- Wrong `DEPLOY_GIT_USER` or `DEPLOY_GIT_REPO` in `.env`
- GitHub PAT expired or missing `repo` scope
- Merge conflicts on the server (fix manually with `git reset --hard origin/main`)

### "command not found" during build

The `node_modules/.bin` PATH fix should handle this. If it persists:
- Verify `npm install` ran successfully (check `node_modules/` exists)
- Check the build script in `package.json` isn't using a global binary

### Build timeout

Increase the timeout in the `runCommand` call for the build step. Default is 300s (5 min). For very large Next.js projects on small VPS, you may need 600s.

## License

MIT
