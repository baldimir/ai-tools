# Alert Fetching Guide

This guide provides detailed instructions for fetching and parsing Dependabot alerts from GitHub repositories, including remote detection and selection for fork workflows.

## Overview

The alert fetching process retrieves all open Dependabot security alerts from a GitHub repository using the GitHub CLI (`gh`) tool. Before fetching alerts, the skill detects all Git remotes and selects TWO appropriate GitHub remotes to support fork workflows:
1. **Alert/PR Target Remote**: Where Dependabot alerts are fetched from and PRs are created to
2. **Push Remote**: Where fix branches are pushed to

The alerts are then parsed and structured for further processing.

## Prerequisites

- GitHub CLI (`gh`) installed and authenticated
- Repository with at least one GitHub remote
- Repository with Dependabot enabled on the selected remote
- Read access to repository security alerts

## Authentication Check

Before fetching alerts, verify GitHub CLI authentication:

```bash
gh auth status
```

**Expected Output:**
```
✓ Logged in to github.com as username (keyring)
✓ Git operations for github.com configured to use https protocol.
✓ Token: *******************
```

**If Not Authenticated:**
```bash
gh auth login
# Follow the interactive prompts
```

## Remote Detection and Selection (Alert & Push)

Before fetching alerts, the skill must identify which GitHub remotes to use for a fork workflow. Repositories can have multiple remotes (origin, upstream, fork, etc.), and the skill needs to:
1. Select the remote where Dependabot alerts are available (typically `upstream` in fork workflows)
2. Select the remote where branches should be pushed (typically `origin` for your fork)

This two-remote approach supports the common fork workflow where:
- Dependabot alerts come from the upstream repository
- Fix branches are pushed to your fork
- PRs are created from your fork to the upstream repository

### Step 1: Detect All Git Remotes

List all configured remotes:

```bash
git remote -v
```

**Example Output:**
```
origin  https://github.com/username/repo.git (fetch)
origin  https://github.com/username/repo.git (push)
upstream        https://github.com/organization/repo.git (fetch)
upstream        https://github.com/organization/repo.git (push)
fork    git@github.com:username/fork.git (fetch)
fork    git@github.com:username/fork.git (push)
gitlab  https://gitlab.com/username/repo.git (fetch)
gitlab  https://gitlab.com/username/repo.git (push)
```

### Step 2: Filter GitHub Remotes

Extract only GitHub remotes (URLs containing github.com):

```bash
# Get unique GitHub remote names
git remote -v | grep github.com | awk '{print $1}' | sort -u
```

**Example Output:**
```
fork
origin
upstream
```

### Step 3: Parse Remote Information

For each GitHub remote, extract the owner/repo path:

```bash
#!/bin/bash

# Function to parse GitHub remote URL
parse_github_remote() {
    local url="$1"
    # Handles both HTTPS and SSH formats
    # HTTPS: https://github.com/owner/repo.git
    # SSH: git@github.com:owner/repo.git
    echo "$url" | sed -E 's#.*[:/]([^/]+)/([^/]+)(\.git)?$#\1/\2#'
}

# Get all GitHub remotes with their owner/repo
for remote in $(git remote -v | grep github.com | awk '{print $1}' | sort -u); do
    url=$(git remote get-url "$remote")
    owner_repo=$(parse_github_remote "$url")
    echo "$remote: $owner_repo ($url)"
done
```

**Example Output:**
```
fork: username/fork (git@github.com:username/fork.git)
origin: username/repo (https://github.com/username/repo.git)
upstream: organization/repo (https://github.com/organization/repo.git)
```

### Step 4: Apply Two-Step Selection Logic

The skill uses a two-step process to select remotes for fork workflows:

#### Case 1: No Remotes Found

```bash
if [ $(git remote | wc -l) -eq 0 ]; then
    echo "Error: No Git remotes found."
    echo "Please add a remote: git remote add origin <url>"
    exit 1
fi
```

#### Case 2: No GitHub Remotes Found

```bash
GITHUB_REMOTES=$(git remote -v | grep github.com | awk '{print $1}' | sort -u)

if [ -z "$GITHUB_REMOTES" ]; then
    echo "Error: No GitHub remotes found."
    echo "This skill requires a GitHub remote with Dependabot enabled."
    echo "Available remotes:"
    git remote -v
    exit 1
fi
```

#### Case 3: Single GitHub Remote (Auto-select for both)

```bash
GITHUB_REMOTE_COUNT=$(echo "$GITHUB_REMOTES" | wc -l)

if [ "$GITHUB_REMOTE_COUNT" -eq 1 ]; then
    ALERT_REMOTE="$GITHUB_REMOTES"
    PUSH_REMOTE="$GITHUB_REMOTES"
    ALERT_REMOTE_URL=$(git remote get-url "$ALERT_REMOTE")
    ALERT_REPOSITORY=$(parse_github_remote "$ALERT_REMOTE_URL")
    PUSH_REPOSITORY="$ALERT_REPOSITORY"
    PUSH_OWNER=$(extract_owner "$PUSH_REPOSITORY")
    FORK_WORKFLOW=false
    
    echo "✓ Using GitHub remote: $ALERT_REMOTE"
    echo "  Repository: $ALERT_REPOSITORY"
    echo "  URL: $ALERT_REMOTE_URL"
    echo "  Workflow: Direct push (same remote for alerts and push)"
fi
```

#### Case 4: Multiple GitHub Remotes (Two-Step Selection)

**Step 1: Select Alert/PR Target Remote**

```bash
if [ "$GITHUB_REMOTE_COUNT" -gt 1 ]; then
    echo "Found multiple GitHub remotes:"
    echo ""
    
    i=1
    for remote in $GITHUB_REMOTES; do
        url=$(git remote get-url "$remote")
        owner_repo=$(parse_github_remote "$url")
        
        # Detect if it's likely a fork
        if [[ "$remote" == "origin" ]]; then
            echo "$i. $remote ($url) [Your fork]"
        elif [[ "$remote" == "upstream" ]]; then
            echo "$i. $remote ($url) [Upstream repository]"
        else
            echo "$i. $remote ($url)"
        fi
        echo "   Repository: $owner_repo"
        i=$((i + 1))
    done
    
    echo ""
    echo "Which remote has Dependabot alerts? (Where should PRs be created?)"
    echo "[Suggest: upstream, origin]"
    
    # Wait for user selection
    # Store selected remote in ALERT_REMOTE
    # Store owner/repo in ALERT_REPOSITORY
fi
```

**Step 2: Select Push Remote**

```bash
# After alert remote is selected
echo ""
echo "Where should I push fix branches?"
echo ""

i=1
for remote in $GITHUB_REMOTES; do
    url=$(git remote get-url "$remote")
    owner_repo=$(parse_github_remote "$url")
    
    if [ "$remote" == "$ALERT_REMOTE" ]; then
        echo "$i. $remote ($url) [Same as alert remote - direct push]"
    elif [[ "$remote" == "origin" ]]; then
        echo "$i. $remote ($url) [Your fork] ⭐ Recommended for fork workflow"
    else
        echo "$i. $remote ($url)"
    fi
    echo "   Repository: $owner_repo"
    i=$((i + 1))
done

echo ""
echo "[Suggest: origin (fork workflow), $ALERT_REMOTE (direct push)]"

# Wait for user selection
# Store selected remote in PUSH_REMOTE
# Store owner/repo in PUSH_REPOSITORY
# Extract owner in PUSH_OWNER

# Determine workflow type
if [ "$ALERT_REMOTE" != "$PUSH_REMOTE" ]; then
    FORK_WORKFLOW=true
    echo ""
    echo "✓ Fork workflow detected"
    echo "  - Fetching alerts from: $ALERT_REPOSITORY"
    echo "  - Pushing branches to: $PUSH_REPOSITORY"
    echo "  - PRs will be created from $PUSH_OWNER:branch to $ALERT_REPOSITORY"
else
    FORK_WORKFLOW=false
    echo ""
    echo "✓ Direct push workflow"
    echo "  - Using remote: $ALERT_REMOTE"
    echo "  - Repository: $ALERT_REPOSITORY"
fi
```

### Step 5: Validate Selected Remotes

After selection, verify both remotes are accessible:

```bash
# Test API access to alert remote
if ! gh api "repos/$ALERT_REPOSITORY" &>/dev/null; then
    echo "Warning: Cannot access repository $ALERT_REPOSITORY"
    echo "Possible reasons:"
    echo "  - Repository does not exist"
    echo "  - You don't have access to this repository"
    echo "  - Authentication issue"
    echo ""
    echo "Checking authentication..."
    gh auth status
    exit 1
fi

echo "✓ Successfully validated access to $ALERT_REPOSITORY"

# Test API access to push remote (if different)
if [ "$FORK_WORKFLOW" = true ]; then
    if ! gh api "repos/$PUSH_REPOSITORY" &>/dev/null; then
        echo "Warning: Cannot access repository $PUSH_REPOSITORY"
        echo "You may not have write access to push branches."
        echo "Checking authentication..."
        gh auth status
        exit 1
    fi
    
    echo "✓ Successfully validated access to $PUSH_REPOSITORY"
    
    # Verify push remote is a fork of alert remote (optional check)
    PUSH_PARENT=$(gh api "repos/$PUSH_REPOSITORY" --jq '.parent.full_name' 2>/dev/null || echo "")
    if [ -n "$PUSH_PARENT" ] && [ "$PUSH_PARENT" != "$ALERT_REPOSITORY" ]; then
        echo "⚠️  Warning: $PUSH_REPOSITORY is not a fork of $ALERT_REPOSITORY"
        echo "   Push repository parent: $PUSH_PARENT"
        echo "   This may cause issues with PR creation."
    fi
fi
```

### Complete Two-Remote Selection Workflow

```bash
#!/bin/bash

select_github_remotes() {
    # 1. Check if any remotes exist
    if [ $(git remote | wc -l) -eq 0 ]; then
        echo "Error: No Git remotes found."
        exit 1
    fi
    
    # 2. Get GitHub remotes
    GITHUB_REMOTES=$(git remote -v | grep github.com | awk '{print $1}' | sort -u)
    
    if [ -z "$GITHUB_REMOTES" ]; then
        echo "Error: No GitHub remotes found."
        exit 1
    fi
    
    # 3. Count GitHub remotes
    GITHUB_REMOTE_COUNT=$(echo "$GITHUB_REMOTES" | wc -l)
    
    # 4. Auto-select if only one
    if [ "$GITHUB_REMOTE_COUNT" -eq 1 ]; then
        ALERT_REMOTE="$GITHUB_REMOTES"
        PUSH_REMOTE="$GITHUB_REMOTES"
        FORK_WORKFLOW=false
        echo "✓ Auto-selected remote: $ALERT_REMOTE"
        echo "  Workflow: Direct push (same remote)"
    else
        # 5. Step 1: Ask user to select alert remote
        echo "Found $GITHUB_REMOTE_COUNT GitHub remotes:"
        echo "$GITHUB_REMOTES" | nl
        echo ""
        echo "Which remote has Dependabot alerts? (Where should PRs be created?)"
        # User selection logic here for ALERT_REMOTE
        
        # 6. Step 2: Ask user to select push remote
        echo ""
        echo "Where should I push fix branches?"
        echo "$GITHUB_REMOTES" | nl
        echo ""
        echo "[Suggest: origin (fork workflow), $ALERT_REMOTE (direct push)]"
        # User selection logic here for PUSH_REMOTE
        
        # 7. Determine workflow type
        if [ "$ALERT_REMOTE" != "$PUSH_REMOTE" ]; then
            FORK_WORKFLOW=true
        else
            FORK_WORKFLOW=false
        fi
    fi
    
    # 8. Get remote details
    ALERT_REMOTE_URL=$(git remote get-url "$ALERT_REMOTE")
    ALERT_REPOSITORY=$(echo "$ALERT_REMOTE_URL" | sed -E 's#.*[:/]([^/]+)/([^/]+)(\.git)?$#\1/\2#')
    
    PUSH_REMOTE_URL=$(git remote get-url "$PUSH_REMOTE")
    PUSH_REPOSITORY=$(echo "$PUSH_REMOTE_URL" | sed -E 's#.*[:/]([^/]+)/([^/]+)(\.git)?$#\1/\2#')
    PUSH_OWNER=$(echo "$PUSH_REPOSITORY" | cut -d'/' -f1)
    
    # 9. Validate access
    if ! gh api "repos/$ALERT_REPOSITORY" &>/dev/null; then
        echo "Error: Cannot access $ALERT_REPOSITORY"
        exit 1
    fi
    
    if [ "$FORK_WORKFLOW" = true ]; then
        if ! gh api "repos/$PUSH_REPOSITORY" &>/dev/null; then
            echo "Error: Cannot access $PUSH_REPOSITORY"
            exit 1
        fi
    fi
    
    # 10. Display summary
    if [ "$FORK_WORKFLOW" = true ]; then
        echo "✓ Fork workflow configured"
        echo "  Alert Remote: $ALERT_REMOTE ($ALERT_REPOSITORY)"
        echo "  Push Remote: $PUSH_REMOTE ($PUSH_REPOSITORY)"
        echo "  PRs will be created from $PUSH_OWNER:branch to $ALERT_REPOSITORY"
    else
        echo "✓ Direct push workflow configured"
        echo "  Remote: $ALERT_REMOTE ($ALERT_REPOSITORY)"
    fi
    
    # Export for use in alert fetching and PR creation
    export ALERT_REMOTE
    export ALERT_REPOSITORY
    export ALERT_REMOTE_URL
    export PUSH_REMOTE
    export PUSH_REPOSITORY
    export PUSH_REMOTE_URL
    export PUSH_OWNER
    export FORK_WORKFLOW
}
```

### Edge Cases

#### Non-GitHub Remotes

The skill ignores non-GitHub remotes (GitLab, Bitbucket, etc.):

```bash
# Only GitHub remotes are considered
git remote -v | grep github.com
```

#### SSH vs HTTPS URLs

The parsing logic handles both formats:

```bash
# HTTPS format
https://github.com/username/repository.git

# SSH format
git@github.com:username/repository.git

# Both parse to: username/repository
```

#### Remote Without Dependabot

If the selected remote doesn't have Dependabot enabled:

```bash
# This will be detected when fetching alerts
gh api "repos/$OWNER_REPO/dependabot/alerts"

# If no alerts endpoint available:
# Error: HTTP 404: Not Found
# Dependabot may not be enabled on this repository
```

#### Authentication Issues

If user doesn't have access to the selected remote:

```bash
# Error: HTTP 403: Forbidden
# You don't have permission to access security alerts for this repository
```

---

## Fetching Alerts

After selecting the appropriate GitHub remote, fetch alerts from that repository.

### Basic Alert Fetch

Use the GitHub API via `gh` CLI to fetch all Dependabot alerts from the alert remote:

```bash
# Use ALERT_REPOSITORY from remote selection
gh api repos/${ALERT_REPOSITORY}/dependabot/alerts --paginate
```

**Note:** The `--paginate` flag is essential for repositories with many alerts. GitHub API returns 30 items per page by default, so without pagination you'll only see the first 30 alerts.

### Filtered Alert Fetch (Open Alerts Only)

Fetch only open alerts using jq filtering:

```bash
gh api repos/${ALERT_REPOSITORY}/dependabot/alerts --paginate \
  --jq '.[] | select(.state == "open")' | jq -s .
```

**Note:** When using `--paginate` with `--jq`, the jq filter is applied to each page separately. The final `| jq -s .` collects all results into a single array.

### Structured Alert Fetch

Fetch alerts with specific fields extracted:

```bash
gh api repos/${ALERT_REPOSITORY}/dependabot/alerts --paginate \
  --jq '.[] | select(.state == "open") | {
    number: .number,
    package: .dependency.package.name,
    ecosystem: .dependency.package.ecosystem,
    manifest_path: .dependency.manifest_path,
    severity: .security_advisory.severity,
    cve_id: .security_advisory.cve_id,
    ghsa_id: .security_advisory.ghsa_id,
    summary: .security_advisory.summary,
    description: .security_advisory.description,
    cvss_score: .security_advisory.cvss.score,
    vulnerable_range: .security_vulnerability.vulnerable_version_range,
    patched_version: .security_vulnerability.first_patched_version.identifier
## Pagination

GitHub's REST API returns results in pages, with a default page size of 30 items. For repositories with many Dependabot alerts, you must use pagination to fetch all alerts.

### Why Pagination Matters

Without pagination:
- Only the first 30 alerts are returned
- Large repositories may have hundreds of alerts
- You'll miss critical vulnerabilities that aren't on the first page

### Using --paginate Flag

The `gh` CLI provides a `--paginate` flag that automatically handles pagination:

```bash
# Without pagination (only first 30 alerts)
gh api repos/owner/repo/dependabot/alerts

# With pagination (all alerts)
gh api repos/owner/repo/dependabot/alerts --paginate
```

### Pagination with jq Filtering

When combining `--paginate` with `--jq`, the jq filter is applied to each page separately. To collect all results into a single array, pipe through `jq -s .`:

```bash
# Correct: Collects all paginated results into one array
gh api repos/owner/repo/dependabot/alerts --paginate \
  --jq '.[] | select(.state == "open")' | jq -s .

# Incorrect: Results remain separated by page
gh api repos/owner/repo/dependabot/alerts --paginate \
  --jq '.[] | select(.state == "open")'
```

### How It Works

1. `--paginate` fetches all pages automatically
2. `--jq '.[] | select(.state == "open")'` filters each page
3. `| jq -s .` collects all filtered results into a single JSON array

### Example: Large Repository

For a repository like `kiegroup/kogito-examples` with many alerts:

```bash
# This command fetches ALL alerts across all pages
gh api repos/kiegroup/kogito-examples/dependabot/alerts --paginate \
  --jq '.[] | select(.state == "open") | {
    number: .number,
    package: .dependency.package.name,
    ecosystem: .dependency.package.ecosystem,
    manifest_path: .dependency.manifest_path,
    severity: .security_advisory.severity,
    cve_id: .security_advisory.cve_id,
    ghsa_id: .security_advisory.ghsa_id,
    summary: .security_advisory.summary,
    vulnerable_range: .security_vulnerability.vulnerable_version_range,
    patched_version: .security_vulnerability.first_patched_version.identifier
  }' | jq -s .
```

### Performance Considerations

- Pagination adds minimal overhead for small repositories
- For large repositories, it's essential to get complete data
- The `gh` CLI handles rate limiting automatically
- Results are streamed, so memory usage remains reasonable

### Best Practice

**Always use `--paginate`** when fetching Dependabot alerts to ensure you don't miss any vulnerabilities, regardless of repository size.

  }' | jq -s .
```

**Important Notes:**
- Always use the `ALERT_REPOSITORY` variable from the alert remote selection step to ensure alerts are fetched from the correct repository (where Dependabot is enabled)
- Always include `--paginate` to fetch all alerts across multiple pages
- When using `--paginate` with `--jq`, pipe the output through `| jq -s .` to collect all paginated results into a single JSON array

## Alert Data Structure

Each alert contains the following key information:

### Alert Object

```json
{
  "number": 1,
  "state": "open",
  "dependency": {
    "package": {
      "ecosystem": "npm",
      "name": "lodash"
    },
    "manifest_path": "package.json",
    "scope": "runtime"
  },
  "security_advisory": {
    "ghsa_id": "GHSA-xxxx-xxxx-xxxx",
    "cve_id": "CVE-2021-23337",
    "summary": "Command Injection in lodash",
    "description": "Detailed description...",
    "severity": "high",
    "cvss": {
      "score": 7.2,
      "vector_string": "CVSS:3.1/AV:N/AC:L/PR:H/UI:N/S:U/C:H/I:H/A:H"
    }
  },
  "security_vulnerability": {
    "package": {
      "name": "lodash"
    },
    "vulnerable_version_range": ">= 4.0.0, < 4.17.21",
    "first_patched_version": {
      "identifier": "4.17.21"
    }
  }
}
```

### Key Fields Explained

| Field | Description | Usage |
|-------|-------------|-------|
| `number` | Alert ID | Reference for tracking and closing alerts |
| `package.name` | Package name | Identify which dependency to update |
| `ecosystem` | Package manager | Determine update strategy (npm, pip, etc.) |
| `manifest_path` | Dependency file | Locate file to modify |
| `severity` | Risk level | Prioritize fixes (critical, high, medium, low) |
| `cve_id` | CVE identifier | Reference in PR description |
| `ghsa_id` | GitHub Security Advisory ID | Reference in PR description |
| `summary` | Brief description | Include in PR title/description |
| `cvss.score` | Severity score | Numeric prioritization (0-10) |
| `vulnerable_range` | Affected versions | Understand scope of vulnerability |
| `patched_version` | Fixed version | Target version for update |

## Parsing Alerts

### Step 1: Fetch Raw Data

```bash
# Store alerts in a variable or file (with pagination)
gh api repos/{owner}/{repo}/dependabot/alerts --paginate | jq -s 'add' > alerts.json
```

**Note:** The `| jq -s 'add'` combines all paginated arrays into a single array before saving to file.

### Step 2: Extract Essential Information

```bash
# Parse with jq to get structured data
jq '.[] | select(.state == "open") | {
  number: .number,
  package: .dependency.package.name,
  ecosystem: .dependency.package.ecosystem,
  severity: .security_advisory.severity,
  patched_version: .security_vulnerability.first_patched_version.identifier
}' alerts.json
```

### Step 3: Group by Package

```bash
# Group alerts by package name
jq 'group_by(.dependency.package.name) | 
    map({
      package: .[0].dependency.package.name,
      ecosystem: .[0].dependency.package.ecosystem,
      alert_count: length,
      alerts: map(.number),
      severities: map(.security_advisory.severity),
      patched_version: (map(.security_vulnerability.first_patched_version.identifier) | max)
    })' alerts.json
```

## Error Handling

### Common Errors and Solutions

#### 1. Authentication Error

**Error:**
```
gh: To use GitHub CLI, please authenticate with `gh auth login`
```

**Solution:**
```bash
gh auth login
# Follow interactive prompts
```

#### 2. Repository Not Found

**Error:**
```
Could not resolve to a Repository with the name 'owner/repo'
```

**Solution:**
- Verify repository name is correct
- Check you have access to the repository
- Ensure repository exists

#### 3. No Dependabot Alerts

**Response:**
```json
[]
```

**Interpretation:**
- No open alerts (good!)
- Or Dependabot not enabled
- Or no vulnerable dependencies detected

**Action:**
```bash
# Check if Dependabot is enabled
gh api repos/{owner}/{repo}/vulnerability-alerts
```

#### 4. API Rate Limit

**Error:**
```
API rate limit exceeded
```

**Solution:**
- Wait for rate limit reset
- Use authenticated requests (should have higher limits)
- Check rate limit status:
```bash
gh api rate_limit
```

## Alert Filtering Strategies

### Filter by Severity

```bash
# Critical only
jq '.[] | select(.security_advisory.severity == "critical")' alerts.json

# High and Critical
jq '.[] | select(.security_advisory.severity == "critical" or 
                 .security_advisory.severity == "high")' alerts.json
```

### Filter by Ecosystem

```bash
# npm only
jq '.[] | select(.dependency.package.ecosystem == "npm")' alerts.json

# Python (pip) only
jq '.[] | select(.dependency.package.ecosystem == "pip")' alerts.json
```

### Filter by Package

```bash
# Specific package
jq '.[] | select(.dependency.package.name == "lodash")' alerts.json
```

### Sort by Severity

```bash
# Sort by CVSS score (highest first)
jq 'sort_by(-.security_advisory.cvss.score)' alerts.json
```

## Determining Repository Information

Repository information is determined during the two-step remote selection phase (see "Remote Detection and Selection (Alert & Push)" section above).

### Extract Owner and Repo from Selected Remotes

```bash
# Get alert remote URL
ALERT_REMOTE_URL=$(git remote get-url "$ALERT_REMOTE")

# Parse owner/repo from URL
# For HTTPS: https://github.com/owner/repo.git
# For SSH: git@github.com:owner/repo.git

# Unified approach (works for both HTTPS and SSH)
ALERT_REPOSITORY=$(echo "$ALERT_REMOTE_URL" | sed -E 's#.*[:/]([^/]+)/([^/]+)(\.git)?$#\1/\2#')

echo "Alert Repository: $ALERT_REPOSITORY"

# Get push remote URL (if different)
if [ "$FORK_WORKFLOW" = true ]; then
    PUSH_REMOTE_URL=$(git remote get-url "$PUSH_REMOTE")
    PUSH_REPOSITORY=$(echo "$PUSH_REMOTE_URL" | sed -E 's#.*[:/]([^/]+)/([^/]+)(\.git)?$#\1/\2#')
    PUSH_OWNER=$(echo "$PUSH_REPOSITORY" | cut -d'/' -f1)
    
    echo "Push Repository: $PUSH_REPOSITORY"
    echo "Push Owner: $PUSH_OWNER"
fi
```

### Using Selected Remotes in Commands

All alert fetching commands should use the alert remote:

```bash
# Fetch alerts from alert remote (where Dependabot is enabled)
gh api "repos/$ALERT_REPOSITORY/dependabot/alerts"

# NOT hardcoded to origin:
# gh api "repos/$(git remote get-url origin | ...)/dependabot/alerts"  # ❌ Wrong
```

All push operations should use the push remote:

```bash
# Push branches to push remote (fork or same as alert remote)
git push "$PUSH_REMOTE" branch-name

# NOT hardcoded to origin:
# git push origin branch-name  # ❌ Wrong (may not be the selected push remote)
```

## Complete Fetch Workflow

### Full Implementation with Remote Selection

```bash
#!/bin/bash

# 1. Select GitHub remotes (alert and push)
select_github_remotes() {
    # Get GitHub remotes
    GITHUB_REMOTES=$(git remote -v | grep github.com | awk '{print $1}' | sort -u)
    
    if [ -z "$GITHUB_REMOTES" ]; then
        echo "Error: No GitHub remotes found"
        exit 1
    fi
    
    GITHUB_REMOTE_COUNT=$(echo "$GITHUB_REMOTES" | wc -l)
    
    if [ "$GITHUB_REMOTE_COUNT" -eq 1 ]; then
        ALERT_REMOTE="$GITHUB_REMOTES"
        PUSH_REMOTE="$GITHUB_REMOTES"
        FORK_WORKFLOW=false
        echo "✓ Auto-selected remote: $ALERT_REMOTE (direct push workflow)"
    else
        echo "Found multiple GitHub remotes:"
        echo "$GITHUB_REMOTES" | nl
        
        # Step 1: Select alert remote
        echo ""
        echo "Which remote has Dependabot alerts?"
        # User selection logic here
        ALERT_REMOTE=$(echo "$GITHUB_REMOTES" | head -1)
        echo "Alert remote: $ALERT_REMOTE"
        
        # Step 2: Select push remote
        echo ""
        echo "Where should I push fix branches?"
        # User selection logic here
        PUSH_REMOTE=$(echo "$GITHUB_REMOTES" | tail -1)
        echo "Push remote: $PUSH_REMOTE"
        
        if [ "$ALERT_REMOTE" != "$PUSH_REMOTE" ]; then
            FORK_WORKFLOW=true
            echo "✓ Fork workflow detected"
        else
            FORK_WORKFLOW=false
            echo "✓ Direct push workflow"
        fi
    fi
    
    ALERT_REMOTE_URL=$(git remote get-url "$ALERT_REMOTE")
    ALERT_REPOSITORY=$(echo "$ALERT_REMOTE_URL" | sed -E 's#.*[:/]([^/]+)/([^/]+)(\.git)?$#\1/\2#')
    
    PUSH_REMOTE_URL=$(git remote get-url "$PUSH_REMOTE")
    PUSH_REPOSITORY=$(echo "$PUSH_REMOTE_URL" | sed -E 's#.*[:/]([^/]+)/([^/]+)(\.git)?$#\1/\2#')
    PUSH_OWNER=$(echo "$PUSH_REPOSITORY" | cut -d'/' -f1)
    
    export ALERT_REMOTE
    export ALERT_REPOSITORY
    export PUSH_REMOTE
    export PUSH_REPOSITORY
    export PUSH_OWNER
    export FORK_WORKFLOW
}

# 2. Main workflow
select_github_remotes

if [ "$FORK_WORKFLOW" = true ]; then
    echo "Fetching alerts from: $ALERT_REPOSITORY (remote: $ALERT_REMOTE)"
    echo "Will push branches to: $PUSH_REPOSITORY (remote: $PUSH_REMOTE)"
else
    echo "Fetching alerts from: $ALERT_REPOSITORY (remote: $ALERT_REMOTE)"
fi

# 3. Check authentication
if ! gh auth status &>/dev/null; then
    echo "Error: GitHub CLI not authenticated"
    echo "Run: gh auth login"
    exit 1
fi

# 4. Fetch alerts from alert remote (with pagination)
ALERTS=$(gh api "repos/$ALERT_REPOSITORY/dependabot/alerts" --paginate --jq '.[] | select(.state == "open")' | jq -s .)

# 5. Check if any alerts found
if [ -z "$ALERTS" ]; then
    echo "No open Dependabot alerts found on $ALERT_REPOSITORY"
    exit 0
fi

# 6. Parse and display summary
echo "$ALERTS" | jq -s '
{
  total: length,
  by_severity: group_by(.security_advisory.severity) |
    map({severity: .[0].security_advisory.severity, count: length}) |
    from_entries,
  by_ecosystem: group_by(.dependency.package.ecosystem) |
    map({ecosystem: .[0].dependency.package.ecosystem, count: length}) |
    from_entries,
  by_package: group_by(.dependency.package.name) |
    map({
      package: .[0].dependency.package.name,
      count: length,
      alerts: map(.number)
    })
}'
```

## Testing Alert Fetching

### Test Commands

```bash
# 1. Test authentication
gh auth status

# 2. Test API access
gh api user

# 3. Test repository access
gh api repos/{owner}/{repo}

# 4. Test alert access (with pagination)
gh api repos/{owner}/{repo}/dependabot/alerts --paginate

# 5. Test with filtering (with pagination)
gh api repos/{owner}/{repo}/dependabot/alerts --paginate --jq '.[] | select(.state == "open") | .number' | jq -s .
```

### Expected Outputs

**No Alerts:**
```json
[]
```

**With Alerts:**
```json
[
  {
    "number": 1,
    "state": "open",
    ...
  },
  {
    "number": 2,
    "state": "open",
    ...
  }
]
```

## Best Practices

1. **Always Check Authentication First**
   - Prevents cryptic errors later
   - Provides clear user guidance

2. **Use Structured Queries**
   - Extract only needed fields
   - Reduces data processing overhead

3. **Handle Empty Results Gracefully**
   - No alerts is a success case
   - Provide clear feedback to user

4. **Cache Alert Data**
   - Avoid repeated API calls
   - Store in temporary file for session

5. **Validate Data Structure**
   - Check for required fields
   - Handle missing patched_version gracefully

6. **Log API Calls**
   - Track rate limit usage
   - Debug issues with API responses

## Troubleshooting

### Issue: Alerts Not Showing

**Check:**
1. Is Dependabot enabled?
   ```bash
   gh api repos/{owner}/{repo}/vulnerability-alerts
   ```

2. Are there actually vulnerabilities?
   - Check GitHub Security tab in web UI

3. Is the repository private?
   - Ensure you have security alert access

### Issue: Incomplete Alert Data

**Check:**
1. API response structure
2. jq query syntax
3. Field availability (some fields may be null)

### Issue: Slow API Response

**Solutions:**
1. Use pagination for large result sets
2. Filter at API level when possible
3. Cache results locally

## Summary

This guide covers:
- ✅ Two-remote detection and selection workflow for fork workflows
- ✅ Handling multiple GitHub remotes (alert and push)
- ✅ Parsing remote URLs (HTTPS and SSH)
- ✅ User interaction for two-step remote selection
- ✅ Fork workflow detection and configuration
- ✅ Fetching alerts using GitHub CLI from alert remote
- ✅ Parsing alert data structure
- ✅ Filtering and grouping strategies
- ✅ Error handling approaches
- ✅ Complete workflow implementation with two-remote selection
- ✅ Testing and troubleshooting

**Key Points:**
- Always detect and select TWO remotes before fetching alerts
- Support fork workflows (origin, upstream, etc.)
- Auto-select same remote for both when only one GitHub remote exists
- Ask user for both remotes when multiple GitHub remotes exist
- Use alert remote for fetching alerts and PR target
- Use push remote for pushing branches
- Detect fork workflow automatically based on remote selection

**Next Steps:**
- Use parsed alerts for user selection (Phase 4)
- Process selected alerts for dependency updates (Phase 6)
- Use push remote for pushing branches (Phase 9 and 10)
- Use alert remote for PR creation with fork workflow support (Phase 9 and 10)

---

*Part of Dependabot Auto-Fix Skill v2.0.0*