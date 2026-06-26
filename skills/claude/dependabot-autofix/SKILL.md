---
name: dependabot-autofix
description: Autonomously fix GitHub Dependabot security alerts by updating dependencies, discovering migration guides, applying code fixes, and creating pull requests. Use when the user asks to fix Dependabot alerts, security vulnerabilities, or dependency updates.
argument-hint: "[all | critical | high | <library-name> | top N]"
allowed-tools: Bash Read Edit Write WebFetch WebSearch
---

# Dependabot Auto-Fix Skill

This skill autonomously fixes GitHub Dependabot security alerts by:
1. Fetching alerts from a GitHub repository
2. Allowing user selection of which alerts to fix
3. Updating dependencies to patched versions
4. Verifying builds and tests pass
5. Discovering migration guides when tests fail
6. Automatically applying code fixes for breaking changes
7. Retrying with iterative fixes (up to 3 attempts)
8. Creating pull requests for successful fixes
9. Creating draft PRs with diagnostics for failed fixes

## Prerequisites

Before using this skill, ensure:
- You are in a Git repository with a remote GitHub origin
- GitHub CLI (`gh`) is installed and authenticated
- Dependabot is enabled on the repository
- You have write access to the repository
- The project has a working build and test setup

## User Arguments

The user may provide arguments: `$ARGUMENTS`

Interpret them as follows:
- `all` or empty: Fix all alerts
- `critical`, `high`, `medium`, `low`: Fix alerts of that severity and above
- A library name (e.g., `lodash`): Fix alerts for that specific library
- `top N` (e.g., `top 5`): Fix the N most severe alerts

---

## Workflow Phases

Execute these phases sequentially. Track state (original branch, selected remotes, alert data, PR results) in memory throughout the session.

---

### Phase 1: Initialize and Verify Prerequisites

**Objective:** Ensure the environment is ready for automated fixes.

**Steps:**
1. Verify this is a Git repository:
   ```bash
   git rev-parse --git-dir
   ```

2. Check GitHub authentication:
   ```bash
   gh auth status
   ```

3. Detect all Git remotes:
   ```bash
   git remote -v
   ```

4. Detect project type and package manager by checking for these files:
   - `package.json` (npm/yarn/pnpm)
   - `requirements.txt`, `Pipfile`, `pyproject.toml` (Python)
   - `pom.xml` (Maven)
   - `build.gradle`, `build.gradle.kts` (Gradle)
   - `Gemfile` (Ruby)
   - `composer.json` (PHP)
   - `go.mod` (Go)
   - `Cargo.toml` (Rust)
   - `*.csproj` (.NET)

5. Identify build and test commands from the project configuration.

6. Store the current branch name:
   ```bash
   git branch --show-current
   ```

**Error Handling:**
- If not in a Git repo: Stop and inform the user.
- If `gh` not authenticated: Tell the user to run `! gh auth login`.
- If no package manager detected: Ask the user to specify the project type.

---

### Phase 1.5: Select GitHub Remotes (Alert & Push)

**Objective:** Identify TWO GitHub remotes for fork workflows:
1. **Alert/PR Target Remote**: Where to fetch Dependabot alerts from and create PRs to
2. **Push Remote**: Where to push fix branches

**Reference:** Read `${CLAUDE_SKILL_DIR}/guides/alert-fetching.md` for detailed remote detection.

**Steps:**

1. **Detect all GitHub remotes:**
   ```bash
   git remote -v | grep github.com | awk '{print $1}' | sort -u
   ```

2. **Parse remote information:**
   For each GitHub remote, extract the owner/repo:
   ```bash
   for remote in $(git remote -v | grep github.com | awk '{print $1}' | sort -u); do
       url=$(git remote get-url "$remote")
       owner_repo=$(echo "$url" | sed -E 's#.*[:/]([^/]+)/([^/]+)(\.git)?$#\1/\2#')
       echo "$remote: $owner_repo"
   done
   ```

3. **Selection logic:**
   - **0 GitHub remotes:** Error - inform the user to add a GitHub remote.
   - **1 GitHub remote:** Auto-select for both alert and push. Inform the user.
   - **2+ GitHub remotes:** Ask the user to select:
     - First: Which remote has Dependabot alerts? (Where should PRs target?)
     - Second: Where should fix branches be pushed? (Recommend `origin` for fork workflows)

4. **Store the selected remotes:**
   - `ALERT_REMOTE` — remote name (e.g., `upstream`)
   - `ALERT_REPOSITORY` — owner/repo (e.g., `org/repo`)
   - `PUSH_REMOTE` — remote name (e.g., `origin`)
   - `PUSH_REPOSITORY` — owner/repo (e.g., `user/repo`)
   - `PUSH_OWNER` — owner name (e.g., `user`)
   - `FORK_WORKFLOW` — true if alert and push remotes differ

5. **Display workflow summary:**
   - Fork workflow: "Fetching alerts from: {ALERT_REPOSITORY}, Pushing branches to: {PUSH_REPOSITORY}"
   - Direct push: "Using remote: {ALERT_REMOTE} ({ALERT_REPOSITORY})"

---

### Phase 2: Fetch Dependabot Alerts

**Objective:** Retrieve all open Dependabot security alerts from GitHub.

**Reference:** Read `${CLAUDE_SKILL_DIR}/guides/alert-fetching.md` for detailed implementation.

**Steps:**
1. Use the selected alert remote's owner/repo.
2. Fetch alerts with pagination:
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
       vulnerable_range: .security_vulnerability.vulnerable_version_range,
       patched_version: .security_vulnerability.first_patched_version.identifier
     }' | jq -s .
   ```

**Important:** Always use `--paginate` and pipe through `| jq -s .` to collect all results.

**Error Handling:**
- No alerts found: Inform user, exit gracefully.
- Dependabot not enabled: Provide instructions to enable.
- Authentication fails: Check permissions for selected remote.

---

### Phase 3: Filter and Group Alerts

**Objective:** Organize alerts and prepare for user selection.

**Steps:**
1. Group alerts by package name (one PR per library).
2. Count alerts per package.
3. Categorize by severity (critical, high, medium, low).
4. Sort packages by total severity score (critical=4, high=3, medium=2, low=1).

---

### Phase 4: Present Options and Get User Selection

**Objective:** Allow the user to choose which alerts to fix.

**Reference:** Read `${CLAUDE_SKILL_DIR}/alert-selection-guide.md` for interaction patterns.

**Present a summary to the user:**
```
Found X Dependabot alerts in this repository:

By Severity:
- Critical: X alerts
- High: X alerts
- Medium: X alerts
- Low: X alerts

Grouped by Library:
1. library-a (3 alerts - 2 critical, 1 high)
2. library-b (1 alert - high)
3. library-c (2 alerts - medium)
```

**If the user provided arguments** (`$ARGUMENTS`), apply the filter automatically and confirm.

**Otherwise, offer selection options:**
1. Fix specific library (provide library name)
2. Fix by severity (critical only, high+critical, all)
3. Fix top N alerts (by severity)
4. Fix all alerts

Ask the user which option they prefer and confirm before proceeding.

---

### Phase 5: Process Each Library (Loop)

**Objective:** For each selected library, create a branch and attempt fixes.

**For each library in selection:**

1. **Create feature branch:**
   ```bash
   git checkout -b dependabot-autofix/{package-name}-$(date +%Y%m%d%H%M%S)
   ```

2. **Collect all alerts for this library:**
   - All alert numbers
   - Target version (highest patched version)
   - Alert details for PR description

3. **Proceed to Phase 6.**

---

### Phase 6: Update Dependency

**Objective:** Update the vulnerable dependency to the patched version.

**Reference:** Read `${CLAUDE_SKILL_DIR}/guides/dependency-update.md` for detailed strategies.

**Steps:**
1. Auto-detect package manager from project files.
2. Identify dependency file(s) for the detected ecosystem.
3. Determine current version and target version (from alert's patched version).
4. **Check if dependency is transitive:**
   - If transitive: First try updating the direct dependency that pulls it in.
   - If that resolves it: Proceed with the direct dependency update.
   - If not: Fall back to override mechanisms.
5. Update the dependency file with the new version.
6. Run the appropriate package manager update command.
7. Verify lock files are updated.
8. Commit changes:
   ```bash
   git add {dependency-files}
   git commit -m "Update {package} to {version} to fix security vulnerabilities"
   ```

**Error Handling:**
- Update command fails: Log error, proceed to Phase 10 (draft PR).
- Version not found: Try latest stable version.
- Lock file conflicts: Attempt automatic resolution.

---

### Phase 7: Run Build and Tests

**Objective:** Verify the dependency update doesn't break the project.

**Reference:** Read `${CLAUDE_SKILL_DIR}/guides/build-test-verification.md` for detailed steps.

**Steps:**

1. **Detect test command** from project configuration (package.json scripts, Makefile, CI config, ecosystem defaults).

2. **Run tests with timeout:**
   ```bash
   timeout 600 {test_command}
   ```

3. **Capture and save output** for analysis.

4. **Evaluate results:**
   - **Tests pass:** Proceed to Phase 9 (Create PR).
   - **Tests fail:** Proceed to Phase 8 (Intelligent Auto-Fix).

---

### Phase 8: Intelligent Auto-Fix (Migration Guide Discovery & Code Modification)

**Objective:** Automatically discover migration guides and apply code fixes when tests fail.

**Reference:**
- Read `${CLAUDE_SKILL_DIR}/guides/migration-guide-discovery.md` for discovery strategies
- Read `${CLAUDE_SKILL_DIR}/guides/code-modification.md` for fix implementation

---

#### Phase 8a: Discover Migration Guides

Use a 4-tier search strategy:

**Tier 1: GitHub Repository**
- Check for CHANGELOG.md, UPGRADING.md, MIGRATION.md in the package's GitHub repo
- Check docs/migration/ directory
- Fetch GitHub release notes
```bash
# Find the package's GitHub repo
# Then check for migration docs:
gh api "repos/{owner}/{repo}/contents/CHANGELOG.md" --jq '.download_url' 2>/dev/null
gh api "repos/{owner}/{repo}/releases" --jq '.[] | select(.tag_name | contains("{version}")) | .body'
```

**Tier 2: Package Registry**
- Check package metadata and README
- Look for project URLs and changelog links

**Tier 3: Official Documentation**
- Use `WebFetch` to check docs.{package}.com, {package}.readthedocs.io, etc.
- Search for migration/upgrade sections

**Tier 4: Community Resources**
- Use `WebSearch` to find Stack Overflow discussions and blog posts about the migration
- Search GitHub Issues for migration-related discussions

---

#### Phase 8b: Parse Migration Guides

1. Extract breaking changes from discovered documentation.
2. Identify code patterns that need to change (imports, API signatures, config).
3. Categorize changes by type:
   - Import/module changes
   - API signature changes
   - Configuration changes
   - Behavior changes

---

#### Phase 8c: Apply Code Modifications

Use Claude Code's `Edit` tool for all code modifications (not sed or manual file editing).

1. **Analyze test failures** to determine which files and patterns need fixing.
2. **Find code locations** using grep/search for package usage.
3. **Apply fixes by type:**
   - **Import fixes:** Update import paths, module names
   - **API signature fixes:** Update function calls, parameters
   - **Configuration fixes:** Update config files
   - **Behavior changes:** Update code expectations
4. **Verify syntax** after each modification.
5. **Commit changes:**
   ```bash
   git add -A
   git commit -m "Attempt {N}: Fix {failure_type} for {package}"
   ```

---

#### Phase 8d: Retry Build and Tests

1. Run tests again after applying fixes.
2. Compare results with the previous run.
3. **Iterative fix strategy (max 3 attempts):**
   - Attempt 1: Import fixes (safest)
   - Attempt 2: Configuration fixes
   - Attempt 3: API signature fixes (most complex)
4. **Check progress after each attempt:**
   - Fewer failures: Continue with next attempt.
   - No progress: Try a different strategy.
   - More failures: Roll back and try alternative approach.
5. **After max attempts:**
   - Tests pass: Proceed to Phase 9 (Create PR).
   - Tests still fail: Proceed to Phase 10 (Create Draft PR).

---

#### Phase 8e: Rollback on Failure

If no progress after 3 attempts:

1. Reset to the dependency update commit:
   ```bash
   git reset --hard {start_commit}
   ```
2. Preserve diagnostic information (test outputs, discovered migration guides, attempted fixes).
3. Proceed to Phase 10 (Create Draft PR).

---

### Phase 9: Create Pull Request

**Objective:** Create a PR for the successful dependency update.

**Reference:**
- Read `${CLAUDE_SKILL_DIR}/guides/pr-creation.md` for PR formatting details
- Read `${CLAUDE_SKILL_DIR}/pr-template.md` for templates
- Read `${CLAUDE_SKILL_DIR}/security-checklist.md` for security verification
- Read `${CLAUDE_SKILL_DIR}/guides/security-sanitization.md` for sanitization rules

**CRITICAL: Sanitize all content before creating the PR.**
- Remove all absolute paths (convert to relative)
- Redact all credentials, tokens, API keys
- Remove private IP addresses and internal hostnames
- Redact environment variable values
- Sanitize URLs with embedded credentials

**Steps:**

1. **Push branch:**
   ```bash
   git push ${PUSH_REMOTE} {branch_name}
   ```

2. **Generate PR description** using the success template from `pr-template.md`:
   - List all fixed alerts with CVE IDs
   - Show version changes
   - Include verification results (sanitized)
   - Add migration notes if applicable

3. **Create PR:**

   For **direct push workflow** (same remote):
   ```bash
   gh pr create \
     --repo ${ALERT_REPOSITORY} \
     --title "[Security] Fix Dependabot alerts for {package-name}" \
     --body "$(cat <<'EOF'
   {sanitized PR description}
   EOF
   )" \
     --label "security,dependencies,automated" \
     --base main
   ```

   For **fork workflow** (different remotes):
   ```bash
   gh pr create \
     --repo ${ALERT_REPOSITORY} \
     --head ${PUSH_OWNER}:{branch_name} \
     --title "[Security] Fix Dependabot alerts for {package-name}" \
     --body "$(cat <<'EOF'
   {sanitized PR description}
   EOF
   )" \
     --label "security,dependencies,automated" \
     --base main
   ```

4. **Record PR details** (number, URL, status).

5. **Proceed to Phase 11.**

---

### Phase 10: Create Draft PR (For Failed Fixes)

**Objective:** Create a draft PR when tests fail, documenting the issue.

**Reference:** Same references as Phase 9.

**CRITICAL: Sanitize ALL diagnostic content before creating the draft PR.** Extra scrutiny is needed because failed test output often contains more sensitive information.

**Steps:**

1. **Push branch:**
   ```bash
   git push ${PUSH_REMOTE} {branch_name}
   ```

2. **Generate draft PR description** using the draft template from `pr-template.md`:
   - Mark as requiring manual review
   - Include attempted changes
   - Document test failures (SANITIZED)
   - Include diagnostic output (SANITIZED)
   - Document all auto-fix attempts and their results
   - Provide recommended next steps
   - Link to migration resources if found

3. **Create draft PR:**

   For **direct push workflow**:
   ```bash
   gh pr create \
     --repo ${ALERT_REPOSITORY} \
     --title "[Security] Fix Dependabot alerts for {package-name} (DRAFT - Manual Review Required)" \
     --body "$(cat <<'EOF'
   {sanitized draft PR description}
   EOF
   )" \
     --label "security,dependencies,automated,needs-work" \
     --draft \
     --base main
   ```

   For **fork workflow**:
   ```bash
   gh pr create \
     --repo ${ALERT_REPOSITORY} \
     --head ${PUSH_OWNER}:{branch_name} \
     --title "[Security] Fix Dependabot alerts for {package-name} (DRAFT - Manual Review Required)" \
     --body "$(cat <<'EOF'
   {sanitized draft PR description}
   EOF
   )" \
     --label "security,dependencies,automated,needs-work" \
     --draft \
     --base main
   ```

4. **Record PR details** (number, URL, draft status).

5. **Proceed to Phase 11.**

---

### Phase 11: Cleanup and Next Alert

**Objective:** Clean up current work and move to the next alert or complete.

**Steps:**

1. **Return to original branch:**
   ```bash
   git checkout {original_branch}
   ```

2. **Update progress tracking:**
   - Mark current library as processed.
   - Record PR created (regular or draft).

3. **Check for more alerts:**
   - If more libraries in selection: Return to Phase 5.
   - If all processed: Proceed to Phase 13.

---

### Phase 13: Generate Final Report

**Objective:** Provide a comprehensive summary of all fixes attempted.

Present the following report to the user:

```
Dependabot Auto-Fix Summary
============================

Total Libraries Processed: X
Total Alerts Addressed: X

Successful Fixes (PRs Created):
1. library-a - PR #123
   - Fixed 3 alerts (2 critical, 1 high)
   - Updated from 1.2.3 to 1.2.5
   - All tests passing
   - URL: https://github.com/owner/repo/pull/123

Failed Fixes (Draft PRs Created):
1. library-c - PR #125 (DRAFT)
   - Attempted to fix 2 alerts (medium)
   - Updated from 3.1.0 to 3.2.0
   - Tests failed - manual review required
   - URL: https://github.com/owner/repo/pull/125

Next Steps:
1. Review and merge PR #123 (library-a)
2. Manually fix and complete PR #125 (library-c)

Statistics:
- Success Rate: X% (N/M libraries)
- Alerts Fixed: X/Y (Z%)
```

---

## Important Guidelines

### Security Sanitization (Mandatory)

Before including ANY content in a PR description:
1. Convert all absolute paths to relative paths.
2. Redact credentials, tokens, API keys, passwords.
3. Remove private IP addresses (10.x, 192.168.x, 172.16-31.x).
4. Redact environment variable values (keep names).
5. Sanitize URLs with embedded credentials.
6. Remove internal hostnames and build server information.

Reference: `${CLAUDE_SKILL_DIR}/guides/security-sanitization.md`

### Git Workflow

- Always create new commits (never amend unless explicitly asked).
- Use descriptive commit messages referencing the package and CVE.
- Always return to the original branch after processing each library.
- Never leave the repository in a broken state.

### Error Recovery

- If any phase fails unexpectedly, return to the original branch.
- Delete the feature branch if PR creation fails.
- Preserve work in a draft PR whenever possible.
- Always inform the user of what happened and what to do next.

---

*Version: 2.0.0*
*Last Updated: 2026-06-26*
