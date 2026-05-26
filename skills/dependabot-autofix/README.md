# Dependabot Auto-Fix Skill

**Version:** 2.0.0
**Last Updated:** 2026-05-26

A Bob skill that autonomously fixes GitHub Dependabot security alerts by updating dependencies, **automatically discovering migration guides, applying code fixes for breaking changes**, verifying builds/tests, and creating pull requests.

---

## 📋 Table of Contents

- [Overview](#-overview)
- [Quick Start](#-quick-start)
- [Prerequisites](#-prerequisites)
- [Usage Examples](#-usage-examples)
- [Understanding the Output](#-understanding-the-output)
- [How Code Fixing Works](#-how-code-fixing-works)
- [Understanding Fix Attempts](#-understanding-fix-attempts)
- [Migration Guide Sources](#-migration-guide-sources)
- [Selection Options](#-selection-options)
- [Supported Ecosystems](#-supported-ecosystems)
- [Troubleshooting](#-troubleshooting)
- [Best Practices](#-best-practices)
- [Security & Privacy](#-security--privacy)
- [Limitations](#-limitations)
- [Advanced Topics](#-advanced-topics)

---

## 🎯 Overview

### What Does This Skill Do?

The Dependabot Auto-Fix skill automates the tedious process of fixing security vulnerabilities in your dependencies. Instead of manually updating each vulnerable package, running tests, and creating pull requests, this skill does it all for you. **The skill intelligently handles breaking changes by discovering migration guides and automatically applying code fixes.**

**The skill:**
1. 🔍 Fetches open Dependabot security alerts from your GitHub repository
2. 🎯 Lets you choose which alerts to fix (by library, severity, or count)
3. 🔧 Updates dependencies to patched versions
4. ✅ Runs your test suite to verify nothing breaks
5. 🤖 Discovers migration guides from 4 tiers of sources when tests fail
6. 🔧 Automatically applies code fixes for breaking changes (imports, APIs, configs)
7. 🔁 Retries up to 3 times with different fix strategies
8. 📝 Creates pull requests for successful fixes
9. 📋 Creates draft PRs with diagnostics when automatic fixes don't resolve all issues

### When Should You Use This Skill?

**Use this skill when:**
- You have multiple Dependabot alerts to address
- You want to quickly fix critical security vulnerabilities
- You need to batch-update dependencies safely
- You want automated verification before merging
- You're dealing with breaking changes that have migration guides
- You want automatic code fixes for common breaking changes (imports, API renames, etc.)

**Don't use this skill when:**
- Your project doesn't have a reliable test suite
- You're dealing with major architectural refactoring beyond dependency updates
- The breaking changes require complex business logic modifications

### Key Features

✅ **Intelligent Selection** - Choose alerts by library name, severity level, top N most critical, or fix all at once
✅ **Multi-Ecosystem Support** - Works with npm, pip, Maven, Gradle, and 7+ other package managers
✅ **Automated Testing** - Runs your test suite to catch breaking changes
✅ **Migration Guide Discovery** - Automatically searches 4 tiers (GitHub, registries, docs, community) for migration guides
✅ **Automatic Code Fixing** - Applies code modifications for breaking changes (imports, API signatures, configs)
✅ **Intelligent Retry Logic** - Up to 3 attempts with progressive fix strategies
✅ **Smart PR Creation** - Creates ready-to-merge PRs for successful fixes, draft PRs with diagnostics for complex issues
✅ **Detailed Reporting** - Provides comprehensive summaries with fix attempts and next steps
✅ **Safe Rollback** - Always returns to your original branch, never leaves repo in broken state

---

## 🚀 Quick Start

### Step 1: Verify Prerequisites

Before using the skill, ensure you have:

```bash
# Check if you're in a Git repository
git rev-parse --git-dir

# Check GitHub CLI authentication
gh auth status

# Verify you're on the correct branch
git branch --show-current
```

### Step 2: Activate the Skill

Navigate to your repository and activate Bob:

```bash
cd /path/to/your/repo
```

Then tell Bob to use the skill:

```
"Use the Dependabot Auto-Fix skill to fix security alerts"
```

Or be more specific:

```
"Fix all critical Dependabot alerts"
"Fix Dependabot alerts for lodash"
"Fix the top 5 most severe alerts"
```

### Step 3: Follow the Interactive Prompts

The skill will:
1. Detect and select GitHub remote (if multiple remotes exist)
2. Show you a summary of all alerts
3. Ask you to select which ones to fix
4. Confirm your selection
5. Process each alert and create PRs

### Step 4: Review and Merge

Check the created PRs on GitHub:
- ✅ **Regular PRs** - Review and merge if tests pass
- 📋 **Draft PRs** - Require manual fixes before merging

---

## ✅ Prerequisites

### Required Tools

| Tool | Purpose | Installation |
|------|---------|--------------|
| **GitHub CLI (`gh`)** | Fetch alerts and create PRs | [`gh auth login`](https://cli.github.com/) |
| **Git** | Version control operations | Pre-installed on most systems |
| **Package Manager** | Update dependencies | npm, pip, maven, etc. |

### Required Permissions

You need:
- ✅ **Read access** to repository security alerts
- ✅ **Write access** to create branches and PRs
- ✅ **Push access** to the repository

### Repository Requirements

Your repository must have:
- ✅ **At least one GitHub remote** - Repository must have a GitHub remote configured
- ✅ **Dependabot enabled** - Check Settings → Security → Dependabot on the target remote
- ✅ **Working test suite** - Tests must run successfully

### Verifying Prerequisites

Run these commands to verify your setup:

```bash
# 1. Check GitHub authentication
gh auth status
# Expected: ✓ Logged in to github.com

# 2. Verify repository has at least one GitHub remote
git remote -v | grep github.com
# Expected: At least one GitHub remote listed

# 3. List all GitHub remotes
git remote -v | grep github.com | awk '{print $1}' | sort -u
# Expected: origin, upstream, fork, etc.

# 4. Check for Dependabot alerts
gh api repos/$(gh repo view --json nameWithOwner -q .nameWithOwner)/dependabot/alerts
# Expected: JSON array of alerts (or empty array if none)

# 5. Verify tests work
npm test  # or your test command
# Expected: Tests pass
```

**If authentication fails:**
```bash
gh auth login
# Follow the interactive prompts to authenticate
```

---

## 🔀 Remote Selection

When your repository has multiple Git remotes (e.g., `origin`, `upstream`, `fork`), the skill needs to know which remotes to use for the fork workflow. The skill selects **TWO remotes**:

1. **Alert/PR Target Remote**: Where Dependabot alerts are fetched from and PRs are created to
2. **Push Remote**: Where fix branches are pushed to

### Why Two-Remote Selection Matters

In typical fork workflows:
- **upstream**: Original repository (`https://github.com/org/repo.git`) - Has Dependabot alerts
- **origin**: Your fork (`https://github.com/yourname/repo.git`) - Where you push branches

The skill needs to:
- Fetch alerts from `upstream` (where Dependabot is enabled)
- Push branches to `origin` (your fork)
- Create PRs from `origin` to `upstream`

This is the standard GitHub fork workflow.

### How It Works

#### Single GitHub Remote (Automatic)

If you have only one GitHub remote, the skill automatically uses it for both operations (direct push workflow):

```
✓ Using GitHub remote: origin
  Repository: user/repo
  URL: https://github.com/user/repo.git
  Workflow: Direct push (same remote for alerts and push)
```

#### Multiple GitHub Remotes (Two-Step Interactive Selection)

If you have multiple GitHub remotes, the skill asks you to select TWO remotes:

**Step 1: Select Alert/PR Target Remote**

```
Found multiple GitHub remotes:

1. origin (https://github.com/user/repo.git) [Your fork]
   Repository: user/repo

2. upstream (https://github.com/org/repo.git) [Main repository]
   Repository: org/repo

Which remote has Dependabot alerts? (Where should PRs be created?)
[Suggest: upstream, origin]
```

**Step 2: Select Push Remote**

```
Where should I push fix branches?

1. origin (https://github.com/user/repo.git) [Your fork] ⭐ Recommended for fork workflow
   Repository: user/repo

2. upstream (https://github.com/org/repo.git) [Same as alert remote - direct push]
   Repository: org/repo

[Suggest: origin (fork workflow), upstream (direct push)]
```

**Selection Tips:**
- **Alert Remote**: Choose where **Dependabot is enabled** (usually `upstream`)
- **Push Remote**: Choose where you have **write access** (usually `origin` for forks)
- **Fork Workflow**: Select different remotes (e.g., alert=upstream, push=origin)
- **Direct Push**: Select same remote for both (e.g., both=origin)

### Fork Workflow Example

**Scenario:** You forked a repository and added both remotes:

```bash
git remote -v
# origin    https://github.com/yourname/repo.git (fetch)
# origin    https://github.com/yourname/repo.git (push)
# upstream  https://github.com/org/repo.git (fetch)
# upstream  https://github.com/org/repo.git (push)
```

**Skill Interaction:**
```
Found 2 GitHub remotes:
1. origin (yourname/repo) [Your fork]
2. upstream (org/repo) [Main repository]

Which remote has Dependabot alerts? (Where should PRs be created?)
> upstream

✓ Alert remote selected: upstream (org/repo)

Where should I push fix branches?
1. origin (yourname/repo) [Your fork] ⭐ Recommended
2. upstream (org/repo) [Same as alert remote]

> origin

✓ Fork workflow detected
  - Fetching alerts from: org/repo
  - Pushing branches to: yourname/repo
  - PRs will be created from yourname:branch to org/repo
```

### Direct Push Workflow Example

**Scenario:** You have write access to the main repository:

```bash
git remote -v
# origin    https://github.com/org/repo.git (fetch)
# origin    https://github.com/org/repo.git (push)
```

**Skill Interaction:**
```
✓ Using GitHub remote: origin
  Repository: org/repo
  Workflow: Direct push (same remote for alerts and push)
```

### What Happens After Selection

#### Fork Workflow (Different Remotes)

1. **Fetching alerts** - `gh api repos/org/repo/dependabot/alerts`
2. **Creating branches** - Branches are created locally
3. **Pushing branches** - `git push origin branch-name`
4. **Creating PRs** - `gh pr create --repo org/repo --head yourname:branch-name`

#### Direct Push Workflow (Same Remote)

1. **Fetching alerts** - `gh api repos/org/repo/dependabot/alerts`
2. **Creating branches** - Branches are created locally
3. **Pushing branches** - `git push origin branch-name`
4. **Creating PRs** - `gh pr create --repo org/repo`

### Troubleshooting Remote Selection

#### No GitHub Remotes Found

**Error:**
```
Error: No GitHub remotes found.
This skill requires a GitHub remote with Dependabot enabled.
```

**Solution:**
```bash
# Add a GitHub remote
git remote add origin https://github.com/owner/repo.git

# Or add upstream
git remote add upstream https://github.com/org/repo.git

# Verify
git remote -v | grep github.com
```

#### Wrong Remote Selected

If you selected the wrong remote, you can:
1. Cancel the current operation
2. Re-run the skill
3. Select the correct remote when prompted

#### Remote Without Dependabot

**Error:**
```
No open Dependabot alerts found on org/repo
```

**Possible Causes:**
- Dependabot not enabled on that repository
- No security vulnerabilities detected
- You don't have access to security alerts

**Solution:**
```bash
# Check if Dependabot is enabled
gh api repos/org/repo/vulnerability-alerts

# Enable Dependabot (requires admin access)
# Go to: Settings → Security → Dependabot alerts → Enable
```

---

## 💡 Usage Examples

### Example 1: Fix All Critical Alerts

**Scenario:** You have multiple critical vulnerabilities and want to fix them all at once.

```
User: "Fix all critical Dependabot alerts"

Bot: Fetching Dependabot alerts...
     
     Found 12 alerts, 3 are critical.
     
     Selected alerts:
     1. library-a (2 critical alerts)
     2. library-b (1 critical alert)
     
     Total: 2 libraries, 3 alerts
     Estimated time: 10-15 minutes
     
     Proceed? (yes/no)

User: "Yes"

Bot: [1/2] Processing library-a (2 alerts)...
       ✓ Updated to 2.1.5
       ✓ Tests passed
       ✓ PR created: #123
     
     [2/2] Processing library-b (1 alert)...
       ✓ Updated to 3.2.0
       ✓ Tests passed
       ✓ PR created: #124
     
     Summary:
     - library-a: PR #123 created ✅
     - library-b: PR #124 created ✅
```

### Example 2: Fix a Specific Library

**Scenario:** You know exactly which library needs updating.

```
User: "Fix Dependabot alerts for library-name"

Bot: Found 3 alerts for library-name:
     - Alert #1: CVE-2021-12345 (critical)
     - Alert #2: CVE-2020-67890 (high)
     - Alert #3: CVE-2019-11111 (critical)
     
     Will update from 1.2.3 to 1.2.5
     
     Proceed? (yes/no)

User: "Yes"

Bot: Processing library-name...
       ✓ Dependency updated
       ✓ Tests passed (127 tests)
       ✓ PR #123 created
     
     PR URL: https://github.com/owner/repo/pull/123
```

### Example 3: Fix Top N Most Critical

**Scenario:** You want to tackle the worst vulnerabilities first.

```
User: "Fix the top 5 most severe Dependabot alerts"

Bot: Selected top 5 alerts by severity:
     1. library-a (2 critical alerts)
     2. library-b (1 high alert)
     3. library-c (1 high alert)
     4. library-d (1 medium alert)
     
     Proceed? (yes/no)

User: "Yes"

Bot: [Processing 4 libraries...]
```

### Example 4: Automatic Code Fixing

**Scenario:** A dependency update introduces breaking changes, but the skill automatically fixes them.

```
User: "Fix library-name alerts"

Bot: Processing library-name...
       ✓ Updated from 1.0.0 to 2.0.0
       ✗ Tests failed (5 failures)
       
       🔍 Discovering migration guides...
       ✓ Found migration guide in GitHub repository
       ✓ Extracted 3 breaking changes
       
       🔧 Attempt 1: Applying import fixes...
       ✓ Fixed 2 import statements
       ✓ Tests: 3 failures remaining (progress!)
       
       🔧 Attempt 2: Applying API signature fixes...
       ✓ Updated 3 function calls
       ✓ Tests: All passing! ✅
       
       ✓ PR #126 created
     
     Summary:
     - Updated library-name from 1.0.0 to 2.0.0
     - Automatically fixed breaking changes:
       • Import path changes (2 files)
       • API signature updates (3 locations)
     - All tests passing (127 tests)
     
     PR URL: https://github.com/owner/repo/pull/126
```

### Example 5: When Automatic Fixes Need Help

**Scenario:** The skill tries to fix breaking changes but needs manual intervention.

```
User: "Fix library-name alerts"

Bot: Processing library-name...
       ✓ Updated from 3.1.0 to 3.2.0
       ✗ Tests failed (4 failures)
       
       🔍 Discovering migration guides...
       ✓ Found UPGRADING.md in GitHub repository
       ✓ Found release notes for 3.2.0
       
       🔧 Attempt 1: Applying configuration fixes...
       ✓ Updated 2 config files
       ✗ Tests: 3 failures remaining
       
       🔧 Attempt 2: Applying API signature fixes...
       ✓ Updated 1 method call
       ✗ Tests: 2 failures remaining
       
       🔧 Attempt 3: Applying behavior change fixes...
       ✓ Updated return value handling
       ✗ Tests: 2 failures remaining (no progress)
       
       ⚠ Creating draft PR with diagnostics...
     
     Draft PR #125 created: [Security] Fix Dependabot alerts for library-name (DRAFT)
     
     Automatic fixes applied (3 attempts):
     - ✓ Configuration updates (2 files)
     - ✓ API signature changes (1 location)
     - ✓ Return value handling (1 location)
     
     Remaining test failures (manual review needed):
     - ApplicationContextTest.testBeanCreation
     - WebConfigTest.testMvcConfig
     
     Resources found:
     - Migration guide: UPGRADING.md (linked in PR)
     - Release notes: v3.2.0 (linked in PR)
     - Stack Overflow discussions (3 relevant posts)
     
     PR URL: https://github.com/owner/repo/pull/125
```

---

## 📊 Understanding the Output

### Progress Updates

During execution, you'll see real-time progress:

```
[1/3] Processing lodash (3 alerts)...
  ✓ Dependency updated to 4.17.21
  ✓ Tests passed (127 tests, 2.3s)
  ✓ PR created: #123

[2/3] Processing requests (1 alert)...
  ✓ Dependency updated to 2.28.0
  ✓ Tests passed (45 tests, 1.1s)
  ✓ PR created: #124

[3/3] Processing spring-core (2 alerts)...
  ✓ Dependency updated to 5.3.20
  ✗ Tests failed (2 failures)
  ⚠ Created draft PR: #125
```

### Status Indicators

| Icon | Meaning |
|------|---------|
| ✓ | Success - operation completed |
| ✗ | Failure - operation failed |
| ⚠ | Warning - manual action needed |
| 🔄 | In progress |
| ℹ️ | Information |

### Final Report

At the end, you'll receive a comprehensive summary:

```
Dependabot Auto-Fix Summary
============================

Total Libraries Processed: 3
Total Alerts Addressed: 6

Successful Fixes (PRs Created):
1. lodash - PR #123 (3 alerts fixed)
   - Updated from 4.17.15 to 4.17.21
   - All tests passing
   - URL: https://github.com/owner/repo/pull/123

2. requests - PR #124 (1 alert fixed)
   - Updated from 2.25.0 to 2.28.0
   - All tests passing
   - URL: https://github.com/owner/repo/pull/124

Failed Fixes (Draft PRs Created):
1. spring-core - PR #125 (2 alerts, manual review required)
   - Updated from 5.3.15 to 5.3.20
   - Tests failed - see PR for diagnostics
   - URL: https://github.com/owner/repo/pull/125

Next Steps:
1. Review and merge PR #123 (lodash)
2. Review and merge PR #124 (requests)
3. Manually fix and complete PR #125 (spring-core)

Statistics:
- Success Rate: 67% (2/3 libraries)
- Alerts Fixed: 4/6 (67%)
- Time Taken: 8 minutes
```

### Understanding PR Types

#### ✅ Regular PR (Tests Passed)

**What it means:** The dependency update was successful and all tests pass.

**What to do:**
1. Review the PR description
2. Check the changes (usually just version bumps)
3. Verify CI/CD passes
4. Merge the PR

**PR includes:**
- List of fixed alerts with CVE IDs
- Version changes
- Test results
- No manual action required

#### 📋 Draft PR (Tests Failed)

**What it means:** The dependency was updated but tests failed. Manual intervention needed.

**What to do:**
1. Read the failure analysis in the PR
2. Check the diagnostic output
3. Review migration guides (if linked)
4. Make necessary code changes
5. Push commits to the PR branch
6. Mark as ready for review once tests pass

**PR includes:**
---

## 🤖 How Code Fixing Works

The skill includes intelligent automatic code fixing when dependency updates introduce breaking changes. Here's how it works:

### The Autonomous Fix Process

When tests fail after a dependency update, the skill automatically:

1. **Discovers Migration Guides** - Searches 4 tiers of sources for migration documentation
2. **Parses Breaking Changes** - Extracts patterns and code modification instructions
3. **Applies Code Fixes** - Makes targeted changes to your codebase
4. **Retries Tests** - Verifies fixes work (up to 3 attempts)
5. **Documents Everything** - Records all attempts in the PR description

### What Types of Changes Can Be Fixed Automatically?

The skill can handle common breaking changes:

✅ **Import/Module Changes**
```javascript
// Before (old version)
import { oldFunction } from 'library';

// After (automatic fix applied)
import { newFunction } from 'library';
```

✅ **API Signature Changes**
```python
# Before (old version)
result = library.process(data)

# After (automatic fix applied)
result = library.process(data, options={})
```

✅ **Configuration Changes**
```javascript
// Before (old version)
const config = { mode: 'legacy' };

// After (automatic fix applied)
const config = { mode: 'modern', compatibility: true };
```

✅ **Renamed Methods/Properties**
```typescript
// Before (old version)
obj.oldMethod();

// After (automatic fix applied)
obj.newMethod();
```

### When Automatic Fixes Don't Work

If the skill can't fully resolve the breaking changes after 3 attempts:
- A **draft PR** is created with all attempted fixes documented
- The PR includes links to migration guides found
- Test failure diagnostics help you complete the fix manually
- All discovered resources are preserved for your review

---

## 🔄 Understanding Fix Attempts

The skill uses a progressive 3-attempt strategy to fix breaking changes:

### Attempt 1: Import and Module Fixes (Safest)

**Focus:** Fix import statements and module references
- Rename imports
- Update module paths
- Fix require statements
- Adjust export statements

**Why first?** These are the safest changes with minimal risk of introducing bugs.

### Attempt 2: Configuration Fixes

**Focus:** Update configuration objects and settings
- Add new required options
- Remove deprecated settings
- Update default values
- Adjust configuration structure

**Why second?** Configuration changes are usually well-documented and straightforward.

### Attempt 3: API Signature Fixes (Most Complex)

**Focus:** Update function calls and method signatures
- Add new required parameters
- Remove deprecated parameters
- Reorder parameters
- Update return value handling

**Why last?** These changes are more complex and require careful analysis of usage patterns.

### Progress Tracking

After each attempt, the skill:
- ✅ Compares test results to previous attempt
- ✅ Checks if fewer tests are failing
- ✅ Commits changes with descriptive messages
- ✅ Decides whether to continue or rollback

**Example progress:**
```
Attempt 1: Import fixes applied
  ✓ Fixed 3 import statements
  ✓ Tests: 45 passing, 2 failing (was 5 failing)
  → Progress detected, continuing...

Attempt 2: Configuration fixes applied
  ✓ Updated 2 config objects
  ✓ Tests: 47 passing, 0 failing
  → All tests passing! ✅
```

---

## 🔍 Migration Guide Sources

The skill searches 4 tiers of sources to find migration documentation:

### Tier 1: GitHub Repository (Primary Source) 🥇

**Most reliable source** - Official documentation from the package maintainers.

**Searches for:**
- `CHANGELOG.md` - Version-specific changes
- `UPGRADING.md` - Upgrade instructions
- `MIGRATION.md` - Migration guides
- `docs/migration/` - Migration documentation directory
- **GitHub Releases** - Release notes with breaking changes

**Example:** For `lodash` update, checks `https://github.com/lodash/lodash/blob/main/CHANGELOG.md`

### Tier 2: Package Registry (Secondary Source) 🥈

**Package-specific metadata** - Information from npm, PyPI, Maven Central, etc.

**Searches for:**
- Package README files
- Homepage URLs
- Documentation links
- Changelog URLs in package metadata

**Example:** For npm packages, checks `npm view lodash` metadata

### Tier 3: Official Documentation Sites 🥉

**Dedicated documentation** - Official docs websites.

**Searches for:**
- `docs.{package}.com`
- `{package}.readthedocs.io`
- `{package}.github.io`
- Migration/upgrade sections

**Example:** For React, checks `https://react.dev/blog` for migration guides

### Tier 4: Community Resources 🌐

**Community knowledge** - When official docs aren't available.

**Searches for:**
- Stack Overflow discussions
- GitHub Issues with migration tags
- Community blog posts (documented for manual review)

**Example:** Finds Stack Overflow posts about migrating from version X to Y

### What Happens When No Guide Is Found?

The skill uses fallback strategies:
1. **Analyzes test failures** - Identifies error patterns (import errors, API changes, etc.)
2. **Searches codebase** - Finds all usage locations of the updated package
3. **Applies conservative fixes** - Makes minimal, safe changes based on error messages
4. **Documents attempts** - Records what was tried in the draft PR

---

- Test failure details
- Error categorization
- Diagnostic output
- Recommended next steps
- Migration guide links

---

## 🎯 Selection Options

The skill offers four flexible ways to select which alerts to fix:

### 1. Fix Specific Library

Target a single package by name.

**Examples:**
- `"Fix library-name"`
- `"Fix Dependabot alerts for package-name"`
- `"Update dependency-name security issues"`

**Best for:** When you know exactly which library needs updating.

### 2. Fix by Severity

Filter alerts by their severity level.

**Examples:**
- `"Fix critical alerts"` - Only critical severity
- `"Fix high and critical"` - High and critical
- `"Fix medium and above"` - Medium, high, and critical

**Severity levels:**
- 🔴 **Critical** - Immediate action required
- 🟠 **High** - Should fix soon
- 🟡 **Medium** - Fix when convenient
- 🟢 **Low** - Fix eventually

**Best for:** Prioritizing by risk level.

### 3. Fix Top N Alerts

Fix a specific number of the most critical alerts.

**Examples:**
- `"Fix top 5 alerts"`
- `"Fix the 3 most critical"`
- `"Fix top 10 most severe"`

**Best for:** Time-boxed security improvements.

### 4. Fix All Alerts

Process every open Dependabot alert.

**Examples:**
- `"Fix all alerts"`
- `"Fix everything"`
- `"Fix all Dependabot security issues"`

**Warning:** For repositories with many alerts (>10), this may take 30-60 minutes.

**Best for:** Comprehensive security cleanup.

---

## 🔧 Ecosystem Support

This skill works with **any package ecosystem that Dependabot supports**. The skill automatically:

- **Detects** the package manager from your project files
- **Uses** the appropriate update commands for that ecosystem
- **Verifies** changes using the ecosystem's build and test tools

**Supported ecosystems include** (but are not limited to):
- Node.js (npm, yarn, pnpm)
- Python (pip, pipenv, poetry)
- Java (Maven, Gradle)
- Ruby (Bundler)
- PHP (Composer)
- Go (Go modules)
- .NET (NuGet)
- Rust (Cargo)
- And any other ecosystem Dependabot supports

### How It Works

The skill uses a **general-purpose approach**:

1. **Auto-detection**: Identifies your package manager from project files
2. **Smart updates**: Uses the correct update command for your ecosystem
3. **Verification**: Runs appropriate build and test commands
4. **Adaptation**: Works with any Dependabot-supported ecosystem without modification

---

## 🔍 Troubleshooting

### Common Issues and Solutions

#### Issue: "gh not authenticated"

**Error message:**
```
Error: gh: not authenticated
```

**Solution:**
```bash
gh auth login
# Follow the interactive prompts
# Choose: GitHub.com → HTTPS → Login with browser
```

**Verify:**
```bash
gh auth status
# Should show: ✓ Logged in to github.com
```

---

#### Issue: "No test command found"

**Error message:**
```
Warning: Could not detect test command
```

**Solution 1 - Add test script to package.json:**
```json
{
  "scripts": {
    "test": "jest"
  }
}
```

**Solution 2 - Specify manually when prompted:**
```
Bot: What command should I use to run tests?
User: "npm test" or "pytest" or "mvn test"
```

---

#### Issue: "Tests failing after update"

**Symptoms:**
- Draft PR created instead of regular PR
- Test failures listed in PR description
- **[Phase 2]** Multiple fix attempts shown in PR

**What Phase 2 Does Automatically:**
The skill now attempts to fix breaking changes automatically:
1. Discovers migration guides from 4 tiers of sources
2. Applies code fixes (imports, API signatures, configs)
3. Retries up to 3 times with different strategies
4. Documents all attempts in the PR

**If automatic fixes succeed:**
- Regular PR is created with all tests passing
- PR description shows what was fixed automatically

**If automatic fixes don't fully resolve issues:**
1. Open the draft PR on GitHub
2. Review the "Automatic Fixes Applied" section
3. Check what the skill already tried (3 attempts documented)
4. Review the "Remaining Test Failures" section
5. Check linked migration guides and resources
6. Make additional code changes locally
7. Push commits to the PR branch

**Common causes that may need manual intervention:**
- **Complex business logic changes** - Requires understanding of application context
- **Behavior changes affecting multiple components** - Cascading changes
- **Custom configuration patterns** - Non-standard setups
- **Missing migration documentation** - No guides available

---

#### Issue: "PR creation failed"

**Error message:**
```
Error: Failed to create pull request
```

**Possible causes and solutions:**

**1. Insufficient permissions:**
```bash
# Check your permissions
gh api repos/{owner}/{repo}/collaborators/{username}/permission
```
Solution: Ask repository admin for write access

**2. Branch protection rules:**
```bash
# Check branch protection
gh api repos/{owner}/{repo}/branches/main/protection
```
Solution: Create PR to a different base branch or adjust protection rules

**3. Branch already exists:**
```bash
# Delete old branch
git push origin --delete dependabot-autofix/lodash-123456
```

**4. Network issues:**
```bash
# Verify connectivity
gh api user
```
Solution: Check internet connection and GitHub status

---

#### Issue: "Dependency update failed"

**Error message:**
```
Error: Failed to update dependency
```

**Possible causes:**

**1. Version not found:**
- The patched version doesn't exist in the registry
- Solution: Check if version is available, try latest stable

**2. Lock file conflicts:**
- Lock file has conflicts after update
- Solution: Delete lock file and regenerate

**3. Package manager error:**
```bash
# For npm
rm -rf node_modules package-lock.json
npm install

# For pip
pip install --upgrade {package}

# For maven
mvn clean install -U
```

---

#### Issue: "Repository not found"

**Error message:**
```
Error: Could not resolve to a Repository
```

**Solution:**
```bash
# Verify remote URL
git remote get-url origin

# Should be: https://github.com/owner/repo.git
# If not, update it:
git remote set-url origin https://github.com/owner/repo.git
```

---

#### Issue: "No GitHub remotes found"

**Error message:**
```
Error: No GitHub remotes found.
This skill requires a GitHub remote with Dependabot enabled.
```

**Cause:**
Your repository has no remotes configured, or all remotes point to non-GitHub hosts (GitLab, Bitbucket, etc.).

**Solution:**
```bash
# Check current remotes
git remote -v

# Add a GitHub remote
git remote add origin https://github.com/owner/repo.git

# Or if you have SSH access
git remote add origin git@github.com:owner/repo.git

# Verify
git remote -v | grep github.com
```

---

#### Issue: "Multiple remotes - which ones to use?"

**Scenario:**
```
Found multiple GitHub remotes:
1. origin (user/repo) [Your fork]
2. upstream (org/repo) [Main repository]

Which remote has Dependabot alerts? (Where should PRs be created?)
```

**How to choose:**

1. **Step 1 - Select Alert Remote (where Dependabot is enabled):**
   ```bash
   # Check origin
   gh api repos/user/repo/dependabot/alerts
   
   # Check upstream
   gh api repos/org/repo/dependabot/alerts
   ```
   
   - If `upstream` has alerts → Select `upstream` as alert remote
   - If `origin` has alerts → Select `origin` as alert remote
   - Usually the **main/original repository**, not your fork

2. **Step 2 - Select Push Remote (where you have write access):**
   
   **For Fork Workflow:**
   - Select `origin` (your fork) as push remote
   - This is the recommended approach for contributors
   
   **For Direct Push:**
   - Select same remote as alert remote (e.g., `upstream`)
   - Only if you have write access to the main repository

3. **Common patterns:**
   - **Forked repo (contributor):** Alert=`upstream`, Push=`origin` (fork workflow)
   - **Your own repo:** Alert=`origin`, Push=`origin` (direct push)
   - **Organization repo (maintainer):** Alert=`origin`, Push=`origin` (direct push)
   - **Organization repo (contributor):** Alert=`upstream`, Push=`origin` (fork workflow)

---

#### Issue: "PR created on wrong repository"

**Problem:**
PR was created on your fork instead of the upstream repository (or vice versa).

**Cause:**
Wrong alert remote was selected during Phase 1.5, or fork workflow not properly detected.

**Solution:**
```bash
# Close the incorrect PR
gh pr close <PR-number>

# Delete the branch from wrong remote
git push <wrong-remote> --delete <branch-name>

# Re-run the skill and select correct remotes
```

**Prevention:**
Always verify the selected remotes before proceeding:
```
✓ Fork workflow detected
  - Fetching alerts from: org/repo  ← Verify this is correct
  - Pushing branches to: yourname/repo  ← Verify this is correct
  - PRs will be created from yourname:branch to org/repo  ← Verify this is correct
```

---

#### Issue: "Cannot push to selected remote"

**Error message:**
```
Error: failed to push some refs to 'upstream'
Permission denied
```

**Cause:**
You don't have write access to the push remote.

**Solution 1 - Use fork workflow:**
```bash
# If you selected 'upstream' as push remote but don't have write access
# Re-run and select:
#   - Alert remote: upstream (where Dependabot is enabled)
#   - Push remote: origin (your fork)
```

**Solution 2 - Fork the repository:**
```bash
# If you don't have a fork yet:
# 1. Fork the repository on GitHub
# 2. Add your fork as a remote:
git remote add origin https://github.com/yourname/repo.git

# 3. Re-run the skill and select:
#   - Alert remote: upstream
#   - Push remote: origin
```

**Solution 3 - Check authentication:**
```bash
# Verify GitHub authentication
gh auth status

# Re-authenticate if needed
gh auth login

# Verify access to push remote
gh api repos/yourname/repo
```

**Solution 4 - Request write access:**
```bash
# If you need direct push access to the main repository
# Ask repository maintainers for write access
```

---

#### Issue: "Dependabot not enabled"


#### Issue: "Migration guide not found" (Phase 2)

**Symptoms:**
- Message: "No migration guide found"
- Skill proceeds with fallback strategies
- Draft PR created with limited fix attempts

**What this means:**
The skill searched all 4 tiers but couldn't find official migration documentation:
- ✗ GitHub repository (no CHANGELOG, UPGRADING.md, etc.)
- ✗ Package registry (no documentation links)
- ✗ Official docs sites (not found or no migration section)
- ✗ Community resources (no relevant discussions)

**What the skill does:**
1. Analyzes test failure patterns
2. Applies conservative fixes based on error messages
3. Documents what was attempted
4. Creates draft PR with diagnostics

**What you can do:**
1. Check the package's GitHub repository manually
2. Search for migration guides on the package's website
3. Look for blog posts about upgrading
4. Check the package's Discord/Slack community
5. Review the test failures to understand what changed
6. Apply fixes manually based on error messages

---

#### Issue: "Automatic fixes introduced new errors" (Phase 2)

**Symptoms:**
- More test failures after fix attempt
- Syntax errors in modified files
- Message: "Rollback performed"

**What happened:**
The skill detected that its changes made things worse and automatically rolled back.

**What the skill does:**
1. Compares test results before and after each fix attempt
2. If more failures occur, rolls back that attempt
3. Tries a different fix strategy
4. After 3 attempts, creates draft PR with diagnostics

**What you can do:**
1. Review the draft PR to see what was attempted
2. Check the "Fix Attempts" section for details
3. The rollback ensures your code is in a clean state
4. You can manually apply fixes based on the diagnostics

---

#### Issue: "Fix attempts exhausted" (Phase 2)

**Symptoms:**
- Message: "Maximum fix attempts reached (3)"
- Draft PR created
- Some progress made but tests still failing

**What this means:**
The skill tried 3 different fix strategies but couldn't fully resolve all issues.

**Review the draft PR:**
1. Check "Automatic Fixes Applied" section - shows what worked
2. Review "Remaining Test Failures" - shows what still needs fixing
3. Check "Migration Resources" - links to guides found
4. Look at "Fix Attempt History" - see what was tried

**Common scenarios:**
- **Partial success:** Some fixes worked, others need manual intervention
- **Complex changes:** Breaking changes require business logic understanding
- **Missing patterns:** Changes don't match common patterns the skill knows

**Next steps:**
1. Build on the automatic fixes already applied
2. Use the migration guides linked in the PR
3. Focus on the remaining test failures
4. Push additional commits to the PR branch

---

**Error message:**
```
No Dependabot alerts found
```

**Solution:**
1. Go to repository Settings on GitHub
2. Navigate to Security & analysis
3. Enable "Dependabot alerts"
4. Enable "Dependabot security updates" (optional)
5. Wait a few minutes for alerts to populate

---

### Getting Additional Help

If you encounter issues not covered here:

1. **Check the detailed guides:**
   - [`guides/alert-fetching.md`](.bob/skills/dependabot-autofix/guides/alert-fetching.md:1) - Alert fetching issues
   - [`guides/dependency-update.md`](.bob/skills/dependabot-autofix/guides/dependency-update.md:1) - Update problems
   - [`guides/build-test-verification.md`](.bob/skills/dependabot-autofix/guides/build-test-verification.md:1) - Test failures
   - [`guides/pr-creation.md`](.bob/skills/dependabot-autofix/guides/pr-creation.md:1) - PR creation issues

2. **Review the technical plan:**
   - [`docs/dependabot-autofix-skill-technical-plan.md`](docs/dependabot-autofix-skill-technical-plan.md:1)

3. **Check supporting files:**
   - [`alert-selection-guide.md`](.bob/skills/dependabot-autofix/alert-selection-guide.md:1) - Selection patterns
   - [`dependency-update-strategies.md`](.bob/skills/dependabot-autofix/dependency-update-strategies.md:1) - Update strategies
   - [`test-verification-checklist.md`](.bob/skills/dependabot-autofix/test-verification-checklist.md:1) - Test verification

---

## 💎 Best Practices

### When to Use Batch Fixes vs Individual Fixes

**Use batch fixes (multiple libraries at once) when:**
- ✅ Alerts are for different, unrelated libraries
- ✅ Updates are minor/patch versions
- ✅ You have a comprehensive test suite
- ✅ You can review multiple PRs efficiently

**Use individual fixes (one library at a time) when:**
- ✅ Dealing with major version upgrades
- ✅ Library has known breaking changes
- ✅ Test suite is incomplete
- ✅ You want to carefully review each change

### Testing Recommendations

**Before running the skill:**
1. ✅ Ensure all tests pass on your current branch
2. ✅ Verify your test suite is comprehensive
3. ✅ Check that tests run in a reasonable time (<10 minutes)

**After PRs are created:**
1. ✅ Review test results in each PR
2. ✅ Run tests locally if you're unsure
3. ✅ Check for any warnings in test output
4. ✅ Verify CI/CD pipelines pass

### PR Review Guidelines

**For successful PRs (tests passed):**
1. ✅ Review the list of fixed alerts
2. ✅ Check version changes are reasonable
3. ✅ Verify no unexpected changes in lock files
4. ✅ Confirm CI/CD passes
5. ✅ Merge promptly to close security alerts

**For draft PRs (tests failed):**
1. ✅ Read the failure analysis carefully
2. ✅ Review diagnostic output
3. ✅ Check migration guides
4. ✅ Make necessary code changes
5. ✅ Test locally before pushing
6. ✅ Update PR description with your changes
7. ✅ Mark as ready for review once tests pass

### Handling Breaking Changes

**Phase 2 handles most breaking changes automatically:**

The skill now automatically:
- ✅ Discovers migration guides from 4 tiers of sources
- ✅ Applies fixes for imports, API signatures, and configurations
- ✅ Retries up to 3 times with different strategies
- ✅ Documents all attempts in the PR

**When automatic fixes succeed:**
- Regular PR is created with all tests passing
- No manual intervention needed
- Review and merge as normal

**When you need to complete the fixes manually:**

1. **Review automatic fixes** - Check what the skill already fixed (documented in PR)
2. **Check the migration guides** - Linked in draft PR with all discovered resources
3. **Review the changelog** - For the new version
4. **Build on existing fixes** - The skill may have fixed 80% already
5. **Test thoroughly** - After each additional change
6. **Document your changes** - Update the PR description

**Common breaking change patterns (now handled automatically):**

| Type | Example | Phase 2 Handling |
|------|---------|------------------|
| **Import paths** | Module moved or renamed | ✅ Automatically fixed (Attempt 1) |
| **API signature** | Function parameters changed | ✅ Automatically fixed (Attempt 3) |
| **Configuration** | Config format changed | ✅ Automatically fixed (Attempt 2) |
| **Removed features** | Method deprecated and removed | ⚠️ May need manual review |
| **Return types** | Function returns different type | ⚠️ May need manual review |
| **Complex logic** | Business logic changes | ⚠️ Requires manual intervention |

### Security Best Practices

1. **Prioritize by severity:**
   - Fix critical alerts immediately
   - Schedule high alerts within a week
   - Plan medium/low alerts for regular maintenance

2. **Don't delay security updates:**
   - Even if tests fail, create the draft PR
   - Work on fixes promptly
   - Don't let security PRs sit unmerged

3. **Review dependencies regularly:**
   - Run this skill weekly or monthly
   - Don't let alerts accumulate
   - Keep dependencies up to date

4. **Understand what you're fixing:**
   - Read the CVE description
   - Understand the vulnerability
   - Know if your code is affected

5. **⚠️ CRITICAL: Review PR content for sensitive information:**
   - All PR content is automatically sanitized
   - Still review PRs before merging
   - Verify no sensitive information leaked
   - See "Security & Privacy" section below

---

## 🔒 Security & Privacy

### Overview

This skill implements **comprehensive security sanitization** to prevent sensitive information leakage in Pull Requests. All PR content is automatically sanitized before creation.

### What Information Is Sanitized?

The skill automatically removes or redacts:

#### 1. **Credentials and Secrets**
- API keys and tokens
- Passwords and passphrases
- OAuth tokens
- SSH keys
- Database credentials
- Service account credentials
- JWT tokens

**Example:**
```
Before: API_KEY=[EXAMPLE_API_KEY]
After:  API_KEY=[REDACTED]
```

#### 2. **Internal File Paths**
- Absolute file paths with usernames
- Home directory paths (`/Users/`, `/home/`, `C:\Users\`)
- System-specific paths

**Example:**
```
Before: /Users/john.doe/projects/myapp/src/main.ts
After:  ./src/main.ts
```

#### 3. **Environment Variables**
- Environment variable values (names are kept)
- Configuration values
- Runtime parameters

**Example:**
```
Before: DATABASE_URL=postgresql://user:pass@localhost:5432/db
After:  DATABASE_URL=[REDACTED]
```

#### 4. **URLs with Authentication**
- URLs containing credentials
- URLs with tokens in query parameters
- Git URLs with embedded credentials

**Example:**
```
Before: https://user:password@api.example.com/data
After:  https://[REDACTED]@api.example.com/data
```

#### 5. **Private Network Information**
- Private IP addresses (10.x, 192.168.x, 172.16-31.x)
- Internal hostnames
- MAC addresses

**Example:**
```
Before: Connecting to 10.0.0.100:5432
After:  Connecting to [PRIVATE_IP]:5432
```

#### 6. **Stack Traces and Error Messages**
- Absolute paths in stack traces (converted to relative)
- Sensitive values in error messages
- Internal debugging information

**Example:**
```
Before: at Database.connect (/Users/john/projects/app/src/db.ts:45:12)
After:  at Database.connect (./src/db.ts:45:12)
```

#### 7. **Build and Test Output**
- Internal build server information
- Test database credentials
- Compilation paths

**Example:**
```
Before: Building on ci-server.example.com
After:  Building on [BUILD_SERVER]
```

### What Information Is Safe to Include?

The following information is **safe** and **included** in PRs:

✅ **Safe Information:**
- Dependency names (e.g., `lodash`, `react`)
- Dependency versions (e.g., `4.17.21`)
- CVE identifiers (e.g., `CVE-2021-23337`)
- Public error types (e.g., `TypeError`)
- Test names (without output values)
- Relative file paths (e.g., `./src/utils.ts`)
- Public documentation URLs
- Generic environment names (e.g., `production`, `staging`)

### Sanitization Process

The skill follows a **multi-layer sanitization process**:

1. **Automatic Pattern-Based Sanitization**
   - Regex patterns detect and redact sensitive information
   - Applied to all test output, build logs, and error messages

2. **Security Checklist Verification**
   - Comprehensive checklist ensures all sensitive patterns are caught
   - Manual review step before PR creation

3. **File Separation**
   - Raw files kept locally for debugging (never included in PR)
   - Sanitized files created specifically for PR inclusion
   - Clear naming: `*-raw.log` vs `*-sanitized.log`

### Security Documentation

For complete details on security sanitization:

- **Sanitization Guide:** `guides/security-sanitization.md`
  - Complete list of sensitive patterns
  - Sanitization strategies
  - Examples of safe vs unsafe content

- **Security Checklist:** `security-checklist.md`
  - Pre-PR creation checklist
  - Verification steps
  - Common mistakes to avoid

### Security Guarantees

✅ **What We Guarantee:**
- All PR content goes through sanitization
- Common sensitive patterns are automatically detected
- Multiple layers of protection
- Clear documentation of what's sanitized

⚠️ **What You Should Still Do:**
- Review PRs before merging
- Verify sanitization was effective
- Report any missed patterns
- Keep security guides updated

### Reporting Security Issues

If you discover sensitive information that wasn't sanitized:

1. **Immediately close the PR** (don't wait)
2. **Rotate any compromised credentials**
3. **Delete the PR branch**
4. **Report the pattern** so it can be added to sanitization rules
5. **Update security guides** with the new pattern

### Privacy Considerations

This skill:
- ✅ **Does NOT** send any data to external services (except GitHub)
- ✅ **Does NOT** store sensitive information
- ✅ **Does** sanitize all public-facing content
- ✅ **Does** keep raw logs local only
- ✅ **Does** provide clear documentation of what's included in PRs

---

## ⚠️ Limitations

### Current Capabilities

The skill provides comprehensive autonomous capabilities:

#### ✅ What IS Included

- ✅ Alert fetching and filtering
- ✅ User selection with 5 options
- ✅ Dependency updates for 8+ ecosystems
- ✅ Build and test verification
- ✅ **Automatic migration guide discovery** (4-tier search strategy)
- ✅ **Automatic code modification** (imports, API signatures, configs)
- ✅ **Intelligent retry logic** (up to 3 attempts with progressive strategies)
- ✅ **Pattern-based code analysis** and fixes
- ✅ **Rollback mechanisms** for failed fixes
- ✅ PR creation for successful fixes
- ✅ Draft PR creation with comprehensive diagnostics
- ✅ Progress tracking and detailed reporting
- ✅ Error handling and safe rollback

#### ⚠️ What Requires Manual Intervention

**Complex breaking changes:**
- Business logic modifications that require domain knowledge
- Architectural changes affecting multiple components
- Custom patterns not matching common migration patterns
- Changes requiring understanding of application context

**Advanced scenarios:**
- Monorepo structures with complex dependencies
- Private package registries with custom authentication
- Highly customized build/test setups
- Transitive dependency conflicts

### General Limitations

**Requires working test suite:**
- If your tests don't work, the skill can't verify fixes
- Flaky tests may cause false failures
- Tests must complete within 10 minutes

**GitHub-specific:**
- Only works with GitHub repositories
- Requires GitHub CLI (`gh`)
- Requires Dependabot to be enabled

**Manual intervention sometimes needed:**
- Breaking changes require manual fixes
- Complex dependency trees may need manual resolution
- Some edge cases aren't handled automatically

**Performance considerations:**
- Processing many alerts takes time (2-5 minutes per library)
- Large test suites slow down verification
- Network speed affects GitHub API calls

### Known Issues

1. **Transitive dependencies:**
   - May not handle complex transitive dependency updates
   - Some indirect dependencies may need manual updates

2. **Monorepos:**
   - Limited support for monorepo structures
   - May need to run skill in each package directory

3. **Private registries:**
   - May have issues with private package registries
   - Authentication for private registries must be pre-configured

4. **Lock file conflicts:**
   - Rare cases of lock file conflicts may occur
   - Usually resolved by deleting and regenerating lock file

---

## 🔬 Advanced Topics

### File Structure

The skill consists of multiple files organized for clarity:

```
.bob/skills/dependabot-autofix/
├── SKILL.md                          # Main skill definition (11 workflow phases)
├── README.md                         # This file
├── alert-selection-guide.md          # User interaction patterns
├── dependency-update-strategies.md   # Quick reference for updates
├── test-verification-checklist.md    # Test verification checklist
├── pr-template.md                    # PR description templates
└── guides/
    ├── alert-fetching.md             # Detailed alert fetching guide
    ├── dependency-update.md          # Detailed update guide
    ├── migration-guide-discovery.md  # [Phase 2] Migration guide discovery
    ├── code-modification.md          # [Phase 2] Automatic code fixing
    ├── build-test-verification.md    # Detailed testing guide
    └── pr-creation.md                # Detailed PR creation guide
```

### Workflow Phases

The skill operates through 11 distinct phases:

1. **Initialize** - Verify prerequisites and detect project type
2. **Fetch Alerts** - Retrieve Dependabot alerts from GitHub
3. **Filter and Group** - Organize alerts by package
4. **User Selection** - Allow user to choose which alerts to fix
5. **Process Each Library** - Create branch for each selected library
6. **Update Dependency** - Update to patched version
7. **Run Tests** - Verify tests pass
8. **🆕 Intelligent Auto-Fix (Phase 2)** - Discover migration guides and apply code fixes
   - 8a. Discover migration guides (4-tier search)
   - 8b. Parse breaking changes and patterns
   - 8c. Apply code modifications
   - 8d. Retry tests (up to 3 attempts)
   - 8e. Rollback if no progress
9. **Create PR** - Create PR for successful fixes
10. **Create Draft PR** - Create draft PR for complex issues
11. **Cleanup and Report** - Return to original branch and generate summary

For detailed information, see [`SKILL.md`](.bob/skills/dependabot-autofix/SKILL.md:1).

### Customization

You can customize behavior by:

1. **Specifying test commands** when prompted
2. **Choosing selection options** that match your workflow
3. **Reviewing and modifying** draft PRs before marking ready
4. **Adjusting retry strategies** - The skill uses progressive strategies automatically

### Future Enhancements

#### Future Enhancements (Planned)
- ⚙️ Configuration file support (.dependabot-autofix.yml)
- ⚡ Parallel processing of multiple alerts
- 📝 Comprehensive logging and audit trails
- 📈 Telemetry and success metrics (optional)
- 🌐 Support for additional ecosystems (Rust, Composer, NuGet)
- 🔍 Advanced filtering and prioritization options
- 🤖 Machine learning-based fix suggestions
- 🔗 Integration with external code analysis tools

---

## 📚 Additional Resources

### Documentation Files

- **Main Skill Definition:** [`SKILL.md`](.bob/skills/dependabot-autofix/SKILL.md:1)
- **Technical Plan:** [`docs/dependabot-autofix-skill-technical-plan.md`](docs/dependabot-autofix-skill-technical-plan.md:1)

### Detailed Guides

- **Alert Fetching:** [`guides/alert-fetching.md`](.bob/skills/dependabot-autofix/guides/alert-fetching.md:1)
- **Dependency Updates:** [`guides/dependency-update.md`](.bob/skills/dependabot-autofix/guides/dependency-update.md:1)
- **Migration Guide Discovery:** [`guides/migration-guide-discovery.md`](.bob/skills/dependabot-autofix/guides/migration-guide-discovery.md:1)
- **Code Modification:** [`guides/code-modification.md`](.bob/skills/dependabot-autofix/guides/code-modification.md:1)
- **Build & Test Verification:** [`guides/build-test-verification.md`](.bob/skills/dependabot-autofix/guides/build-test-verification.md:1)
- **PR Creation:** [`guides/pr-creation.md`](.bob/skills/dependabot-autofix/guides/pr-creation.md:1)

### Supporting Files

- **Alert Selection Guide:** [`alert-selection-guide.md`](.bob/skills/dependabot-autofix/alert-selection-guide.md:1)
- **Update Strategies:** [`dependency-update-strategies.md`](.bob/skills/dependabot-autofix/dependency-update-strategies.md:1)
- **Test Checklist:** [`test-verification-checklist.md`](.bob/skills/dependabot-autofix/test-verification-checklist.md:1)
- **PR Template:** [`pr-template.md`](.bob/skills/dependabot-autofix/pr-template.md:1)

---

## 📝 Version History

### v2.0.0 (2026-05-26)

**Major release with autonomous code-fixing capabilities:**
- ✅ **Automatic migration guide discovery** - 4-tier search strategy (GitHub, registries, docs, community)
- ✅ **Automatic code modification** - Fixes imports, API signatures, and configurations
- ✅ **Intelligent retry logic** - Up to 3 attempts with progressive strategies
- ✅ **Pattern-based analysis** - Extracts and applies breaking change patterns
- ✅ **Rollback mechanisms** - Safe rollback when fixes don't improve results
- ✅ **Enhanced PR descriptions** - Documents all fix attempts and discovered resources
- ✅ Alert fetching using GitHub CLI
- ✅ User interaction with 5 selection options
- ✅ Dependency updates for 8+ ecosystems
- ✅ Build and test verification
- ✅ PR creation for successful fixes
- ✅ Draft PR creation with diagnostics for failures
- ✅ Progress tracking and reporting
- ✅ Error handling and safe rollback

---

## 🤝 Contributing

To enhance this skill:

1. **Add ecosystem support** - Implement update strategies for new package managers
2. **Improve test failure analysis** - Better categorization and diagnostics
3. **Enhance migration guide discovery** - Improve automatic discovery and parsing
4. **Expand automatic code fixes** - Handle more types of breaking changes
5. **Enhance error messages** - More helpful guidance for users

---

## 📄 License

This skill is part of the Bob AI assistant framework.

---

**Need help?** Review the [Troubleshooting](#-troubleshooting) section or check the detailed guides in the [`guides/`](.bob/skills/dependabot-autofix/guides/) directory.

**Found a bug?** The skill is in active development. Feedback and contributions are welcome!