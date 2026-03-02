---
name: git-ghosthunter
version: 1.0.0
description: Track down and eliminate zombie branches and stale PRs
author: SMOUJBOT
tags: [git, cleanup, branches, pr, repositories]
type: devops
dependencies:
  - git >= 2.30
  - gh (GitHub CLI) >= 2.0
  - jq (for JSON parsing)
  - bash >= 4.0
env_vars:
  - GITHUB_TOKEN (optional, for API calls)
  - GIT_GHOSTHUNTER_DRY_RUN (default: "false")
  - GIT_GHOSTHUNTER_AGE_DAYS (default: "90")
  - GIT_GHOSTHUNTER_PROTECTED_BRANCHES (default: "main,master,develop,production")
---

# Git Ghosthunter

**Purpose**: Detect, analyze, and eliminate zombie branches and stale PRs that clutter repositories, reduce maintenance overhead, and create merge conflicts. This skill automates cleanup of abandoned code that never made it to production.

## Real Use Cases

1. **Zombie Branch Detection**: Find feature branches untouched for 90+ days that could be safely deleted
2. **Stale PR Cleanup**: Identify PRs with no activity for 60+ days and close/clean them
3. **Repository Hygiene**: Weekly cleanup jobs for teams with high branch turnover
4. **Pre-Merge Preparation**: Clean up before major releases to reduce branch chaos
5. **Audit Compliance**: Generate reports of abandoned work for management

## Scope

This skill operates on local and remote git repositories. It does NOT:
- Modify protected branches
- Delete branches with open, active PRs (unless specified)
- Touch tags or releases
- Rewrite history

Commands executed:
- `git branch --list --all`
- `git for-each-ref` with date filtering
- `git log -1 --format=%ci <branch>`
- `gh pr list` with JSON output
- `git branch -d` / `-D` for deletion
- `gh pr close` for PR cleanup
- `git push origin --delete` for remote branch removal

## Work Process

### Phase 1: Discovery
```bash
# Collect all local branches with last commit dates
git for-each-ref --format='%(refname:short)|%(committerdate:iso8601)|%(committerdate:unix)' refs/heads/ > branches.txt

# Collect all remote branches
git for-each-ref --format='%(refname:short)|%(committerdate:iso8601)|%(committerdate:unix)' refs/remotes/origin/ > remote_branches.txt
```

### Phase 2: Identify Zombies
```bash
CUTOFF=$(date -d "-${GIT_GHOSTHUNTER_AGE_DAYS:-90} days" +%s)
while IFS='|' read branch date timestamp; do
  if [ "$timestamp" -lt "$CUTOFF" ]; then
    echo "ZOMBIE: $branch (last commit: $date)"
  fi
done < branches.txt
```

### Phase 3: PR Correlation
```bash
# Check if zombie branch has open PR
gh pr list --json headRefName,state,updatedAt,url | jq -r '.[] | select(.headRefName=="'"$branch"'") | "\(.url)|\(.updatedAt)"'
```

### Phase 4: Classification
Branch classified as:
- **SAFE_DELETE**: No open PR, no recent activity, not protected
- **STALE_PR**: PR exists but no updates > 60 days
- **PROTECTED**: In protected branches list
- **ACTIVE**: Recent commits (< age threshold)

### Phase 5: Action Execution
```bash
# For SAFE_DELETE branches:
git branch -d "$branch" 2>/dev/null || git branch -D "$branch"
git push origin --delete "$branch"

# For STALE_PR:
gh pr close "$pr_number" --comment "Closing stale PR after ${GIT_GHOSTHUNTER_AGE_DAYS} days of inactivity"
```

### Phase 6: Reporting
Generate SKILL_REPORT.md with:
- Total branches scanned
- Zombies found
- Deletions performed
- PRs closed
- Errors encountered

## Golden Rules

1. **NEVER delete protected branches**: Check against `$GIT_GHOSTHUNTER_PROTECTED_BRANCHES` before any deletion
2. **Dry-run by default**: Set `GIT_GHOSTHUNTER_DRY_RUN=true` to preview actions
3. **PRs require extra caution**: Only auto-close PRs if flag `--force-pr-cleanup` passed
4. **Backup first**: Create backup branch tags before mass deletion
5. **Remote before local**: Delete remote branch first, then local
6. **Respect worktrees**: Don't delete branches currently checked out in any worktree
7. **No force without confirmation**: `-D` (force delete) requires explicit `--allow-force` flag
8. **Log everything**: Every action written to `$WORKDIR/.git-ghosthunter.log`

## Examples

**Example 1: Find zombie branches only (dry-run)**
```
Input: /ghosthunter scan --dry-run --age 60
Output:
[+] Scanning repository...
[+] Total branches: 47
[+] Zombie branches detected: 8
[+] Protected branches: 3 (main, develop, staging)
[+] Safe to delete: 5
[+] DRY-RUN - No changes made
[+] Run again without --dry-run to delete
Report: SKILL_REPORT_20240115_143022.md
```

**Example 2: Full cleanup including stale PRs**
```
Input: /ghosthunter full-cleanup --age 90 --force-pr-cleanup --allow-force
Output:
[+] Backup tags created: ghosthunter-backup-20240115-1430
[+] Deleting remote: feature/old-login-refactor (zombie: 156 days)
[+] Deleting local: feature/old-login-refactor
[+] Closing PR #342: "Old login refactor" (stale: 89 days)
[+] Cleanup complete: 12 branches removed, 4 PRs closed
Report: SKILL_REPORT_20240115_143045.md
```

**Example 3: Generate zombie report only**
```
Input: /ghosthunter report --format json --age 120
Output:
{
  "repository": "myapp/frontend",
  "scan_time": "2024-01-15T14:30:45Z",
  "total_branches": 52,
  "zombie_branches": [
    {
      "name": "feature/dark-mode-v1",
      "last_commit": "2023-08-20T10:15:00Z",
      "days_inactive": 148,
      "has_open_pr": false,
      "protected": false
    }
  ],
  "stale_prs": 3,
  "protected_branches": ["main", "develop"]
}
```

**Example 4: Interactive cleanup mode**
```
Input: /ghosthunter interactive
Output:
? Select branches to delete (use space to toggle):
  [ ] feature/experiment-api (142 days) - No PR
  [ ] feature/old-dashboard (98 days) - PR #289
  [✓] feature/broken-refactor (180 days) - No PR
  [ ] feature/2023-q4-goals (200+ days) - PR #156
? Confirm deletion of 1 selected branch? (y/N)
```

**Example 5: Repository-wide audit**
```
Input: /ghosthunter audit --all-repos --output ghosthunter-audit-$(date +%Y%m).csv
Output:
[+] Scanning 23 repositories...
[+] Complete. Results saved to ghosthunter-audit-202401.csv
CSV Columns: repository,branch,last_commit_days,has_pr,pr_number,pr_days_inactive,protected,recommended_action
```

## Environment Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `GIT_GHOSTHUNTER_DRY_RUN` | `false` | Set to `true` to preview only |
| `GIT_GHOSTHUNTER_AGE_DAYS` | `90` | Inactivity threshold for zombie classification |
| `GIT_GHOSTHUNTER_PR_AGE_DAYS` | `60` | Inactivity threshold for PR closure |
| `GIT_GHOSTHUNTER_PROTECTED_BRANCHES` | `main,master,develop,production` | Comma-separated protected branch names |
| `GIT_GHOSTHUNTER_BACKUP_PREFIX` | `ghosthunter-backup` | Prefix for backup tags |
| `GITHUB_TOKEN` | (unset) | Optional: use for API rate limit increases |
| `GIT_GHOSTHUNTER_LOG_LEVEL` | `INFO` | Logging: DEBUG, INFO, WARN, ERROR |

## Dependencies

### Required
- `git` >= 2.30 (for `for-each-ref` formatting)
- `bash` >= 4.0 (for associative arrays)
- `jq` >= 1.6 (for PR JSON parsing)
- `gh` (GitHub CLI) >= 2.0 (for PR operations)

### Optional
- `csvkit` (for CSV report formatting)
- `column` (for pretty table output)

### Install on Ubuntu/Debian
```bash
apt-get update && apt-get install -y git jq gh
```

### Install on macOS
```bash
brew install git jq gh
```

## Verification

After cleanup, verify:
```bash
# Confirm branches removed locally
git branch --list | grep -E "feature/|bugfix/|hotfix/"

# Confirm remote branches removed
git ls-remote --heads origin | grep -E "feature/|bugfix/"

# Check for remaining stale PRs
gh pr list --state open --json updatedAt,headRefName | jq -r '.[] | select(.updatedAt < "$(date -d "60 days ago" -Iseconds)")'

# Verify protected branches intact
git branch --list | grep -E "^(main|master|develop|production)$"
```

## Rollback

If deletion was mistake, immediately:

```bash
# List backup tags created before deletion
git tag -l "ghosthunter-backup-*" | sort -r

# Restore specific branch from backup
git checkout -b feature/branch-name ghosthunter-backup-20240115-1430/feature/branch-name

# Push restored branch to remote
git push origin feature/branch-name

# For PR restoration: cannot auto-reopen closed PRs
# Manual step: Create new PR with same branch, reference old PR in description
gh pr create --fill --body "Restored from ghosthunter backup. Original PR #342."
```

**Rollback Complete**: Branch restored, PR must be manually recreated.

## Troubleshooting

**Issue**: "error: branch not found" during deletion
- **Cause**: Branch already deleted (possibly by another process)
- **Fix**: Continue, log as warning. Inspect `SKILL_REPORT.log` for race conditions

**Issue**: "Authentication failed" with gh CLI
- **Cause**: Not logged in or expired token
- **Fix**: `gh auth refresh` or `gh auth login`

**Issue**: "protected branch" errors
- **Cause**: Branch matches protected list but not in local config
- **Fix**: Verify `GIT_GHOSTHUNTER_PROTECTED_BRANCHES` includes it

**Issue**: Deleting branch in current worktree
- **Cause**: Branch checked out in another worktree location
- **Fix**: `git worktree list` to find and switch, or use `--force` carefully

**Issue**: PR still open after cleanup
- **Cause**: PR updated too recently, or different headRefName
- **Fix**: Manual review: `gh pr view <number>` and close manually

## Performance Notes

- Large repos (>500 branches): Use `--batch-size 50` to chunk processing
- Network-bound: GH API calls throttled to 30 req/min (auto-handled)
- Memory: < 100MB even for 1000+ branches (streaming JSON parse)

```