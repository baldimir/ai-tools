---
name: dependabot-autofix
description: Autonomously fix GitHub Dependabot security alerts by updating dependencies, discovering migration guides, applying code fixes, and creating pull requests
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

## Workflow Phases

This skill operates through 13 distinct phases in a state machine workflow:

### Phase 1: Initialize and Verify Prerequisites

**Objective:** Ensure the environment is ready for automated fixes.

**Steps:**
1. Verify we're in a Git repository
2. Check GitHub authentication (`gh auth status`)
3. Detect all Git remotes
4. Verify Dependabot is enabled (check for alerts)
5. Detect project type and package manager
6. Identify build and test commands
7. Store the original branch name for later restoration

**Commands:**
```bash
# Check git repository
git rev-parse --git-dir

# Check GitHub auth
gh auth status

# List all remotes
git remote -v

# Detect package manager from project files
ls package.json requirements.txt pom.xml build.gradle Gemfile composer.json go.mod Cargo.toml
```

**Error Handling:**
- If not in a Git repo: Exit with clear error message
- If `gh` not authenticated: Provide authentication instructions
- If no package manager detected: Ask user to specify project type
- If no remotes found: Exit with error message

---

### Phase 1.5: Select GitHub Remotes (Alert & Push)

**Objective:** Identify and select TWO GitHub remotes to support fork workflows:
1. **Alert/PR Target Remote**: Where to fetch Dependabot alerts and create PRs to
2. **Push Remote**: Where to push fix branches

**Reference:** See `guides/alert-fetching.md` for detailed remote detection and selection.

**Steps:**

1. **Detect All Git Remotes:**
   ```bash
   # List all remotes with URLs
   git remote -v
   ```

2. **Filter GitHub Remotes:**
   ```bash
   # Extract GitHub remotes only (containing github.com)
   git remote -v | grep github.com | awk '{print $1}' | sort -u
   ```

3. **Parse Remote Information:**
   ```bash
   # For each GitHub remote, get owner/repo
   for remote in $(git remote -v | grep github.com | awk '{print $1}' | sort -u); do
       url=$(git remote get-url "$remote")
       # Parse owner/repo from URL (handles both HTTPS and SSH)
       owner_repo=$(echo "$url" | sed -E 's#.*[:/]([^/]+)/([^/]+)(\.git)?$#\1/\2#')
       echo "$remote: $owner_repo"
   done
   ```

4. **Step 1: Select Alert/PR Target Remote:**
   - **0 GitHub remotes:** Error - "No GitHub remotes found. Please add a GitHub remote."
   - **1 GitHub remote:** Auto-select and inform user
   - **2+ GitHub remotes:** Ask user to select

5. **User Interaction for Alert Remote (Multiple Remotes):**
   ```
   Found multiple GitHub remotes:
   1. origin (https://github.com/user/repo.git) [Your fork]
   2. upstream (https://github.com/org/repo.git) [Main repository]
   
   Which remote has Dependabot alerts? (Where should PRs be created?)
   [Suggest: upstream, origin]
   ```

6. **Step 2: Select Push Remote:**
   After alert remote is selected, determine where to push branches:
   
   ```
   Where should I push fix branches?
   1. origin (https://github.com/user/repo.git) [Your fork] ⭐ Recommended for fork workflow
   2. upstream (https://github.com/org/repo.git) [Same as alert remote - direct push]
   
   [Suggest: origin (fork workflow), upstream (direct push)]
   ```

7. **Push Remote Selection Logic:**
   - **Same as alert remote:** Direct push workflow (user has write access)
   - **Different remote (typically origin):** Fork workflow (push to fork, PR to upstream)
   - **Default suggestion:** `origin` if it exists and differs from alert remote
   - **Fallback:** Same as alert remote if only one remote or user has write access

8. **Store Selected Remotes:**
   - **Alert Remote:**
     - Remote name (e.g., "upstream")
     - Remote URL
     - Owner/repo path (e.g., "org/repo")
   - **Push Remote:**
     - Remote name (e.g., "origin")
     - Remote URL
     - Owner/repo path (e.g., "user/repo")
     - Owner name (e.g., "user") - for PR head reference

**Remote URL Parsing:**

Handles both HTTPS and SSH formats:
```bash
# HTTPS: https://github.com/owner/repo.git
# SSH: git@github.com:owner/repo.git

# Unified parsing approach
parse_github_remote() {
    local url="$1"
    echo "$url" | sed -E 's#.*[:/]([^/]+)/([^/]+)(\.git)?$#\1/\2#'
}

# Extract owner from owner/repo
extract_owner() {
    local owner_repo="$1"
    echo "$owner_repo" | cut -d'/' -f1
}
```

**Fork Workflow Detection:**

```bash
# Detect if using fork workflow
if [ "$ALERT_REMOTE" != "$PUSH_REMOTE" ]; then
    echo "✓ Fork workflow detected"
    echo "  - Fetching alerts from: $ALERT_REPOSITORY"
    echo "  - Pushing branches to: $PUSH_REPOSITORY"
    echo "  - PRs will be created from $PUSH_OWNER:branch to $ALERT_REPOSITORY"
    FORK_WORKFLOW=true
else
    echo "✓ Direct push workflow"
    echo "  - Using remote: $ALERT_REMOTE"
    echo "  - Repository: $ALERT_REPOSITORY"
    FORK_WORKFLOW=false
fi
```

**Error Handling:**
- No remotes exist: Exit with instructions to add remote
- No GitHub remotes: Exit with instructions to add GitHub remote
- Remote without Dependabot: Handle gracefully in Phase 2
- Authentication issues: Handle in Phase 2 when fetching alerts
- No write access to push remote: Verify permissions before proceeding
- Push remote not a fork: Warn user but allow (may be intentional)

**Output:**
- Alert remote name stored for use in Phase 2 (alert fetching)
- Push remote name stored for use in Phases 9 and 10 (branch pushing)
- Alert repository (owner/repo) stored for API calls and PR target
- Push repository (owner/repo) stored for PR head reference
- Push owner stored for fork workflow PR creation

---

### Phase 2: Fetch Dependabot Alerts

**Objective:** Retrieve all open Dependabot security alerts from GitHub using the selected remote.

**Reference:** See `guides/alert-fetching.md` for detailed implementation.

**Steps:**
1. Use the selected remote's owner/repo from Phase 1.5
2. Use GitHub CLI to fetch alerts via API
3. Parse JSON response into structured data
4. Extract key information for each alert:
   - Alert number
   - Package name and ecosystem
   - Current vulnerable version
   - Patched version
   - CVE/GHSA ID
   - Severity level
   - Summary description
5. Handle API errors gracefully

**Command:**
```bash
# Use owner/repo from selected remote
OWNER_REPO="${SELECTED_REMOTE_OWNER_REPO}"

gh api repos/${OWNER_REPO}/dependabot/alerts --paginate \
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

**Note:** The `--paginate` flag ensures all alerts are fetched across multiple pages (GitHub API returns 30 items per page by default). The final `| jq -s .` collects all paginated results into a single JSON array.

**Error Handling:**
- If remote has no Dependabot alerts: Inform user, exit gracefully
- If Dependabot not enabled on remote: Provide instructions to enable
- If authentication fails: Check permissions for selected remote
- If API errors: Retry with exponential backoff

**Output:** Array of alert objects for processing.

---

### Phase 3: Filter and Group Alerts

**Objective:** Organize alerts and prepare for user selection.

**Steps:**
1. Group alerts by package name (one PR per library)
2. Count alerts per package
3. Categorize by severity (critical, high, medium, low)
4. Sort packages by total severity score

**Grouping Logic:**
```
For each alert:
  - Add to group by package name
  - Track: alert numbers, severities, CVE IDs
  - Calculate group priority (sum of severity scores)

Sort groups by priority (critical > high > medium > low)
```

---

### Phase 4: Present Options and Get User Selection

**Objective:** Allow user to choose which alerts to fix.

**Reference:** See `alert-selection-guide.md` for interaction patterns.

**Present Summary:**
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

**Offer Selection Options:**
1. Fix specific library (provide library name)
2. Fix by severity (critical only, high+critical, all)
3. Fix top N alerts (by severity)
4. Fix all alerts

**Get User Confirmation:**
- Display selected alerts with details
- Ask for confirmation before proceeding
- Allow user to modify selection

---

### Phase 5: Process Each Library (Loop)

**Objective:** For each selected library, create a branch and attempt fixes.

**For each library in selection:**

1. **Create Feature Branch:**
   ```bash
   git checkout -b dependabot-autofix/{package-name}-{timestamp}
   ```

2. **Get All Alerts for This Library:**
   - Collect all alert numbers
   - Identify target version (highest patched version)
   - Prepare alert details for PR description

3. **Proceed to Phase 6** (Update Dependency)

---

### Phase 6: Update Dependency

**Objective:** Update the vulnerable dependency to the patched version.

**Reference:** See `guides/dependency-update.md` for detailed update strategies.

**Steps:**
1. Auto-detect package manager from project files
2. Identify dependency file(s) for the detected ecosystem
3. Determine current version
4. Determine target version (from alert's patched version)
5. **Check if dependency is transitive:**
   - If transitive: First attempt to update the direct dependency that brought it in
   - Identify the parent/direct dependency
   - Check if updating the direct dependency resolves the vulnerability
   - If successful: Proceed with direct dependency update
   - If unsuccessful: Fall back to using override mechanisms
6. Update dependency file with new version
7. Run appropriate package manager update command
8. Verify lock files are updated (if applicable)
9. Commit changes

**General Update Approach:**

The skill automatically detects the package manager and uses the appropriate update command:

```bash
# Auto-detect package manager
detect_package_manager

# Update using detected package manager's command
update_dependency {package} {version}

# Commit changes
git add {dependency-files}
git commit -m "Update {package} to {version} to fix security vulnerabilities"
```

**Example workflow (ecosystem-agnostic):**
```bash
# 1. Detect ecosystem from project files
# 2. Use appropriate update command for that ecosystem
# 3. Verify changes
# 4. Commit with descriptive message
```

**Error Handling:**
- If update command fails: Log error, skip to Phase 10 (rollback)
- If version not found: Try latest stable version
- If lock file conflicts: Resolve automatically or skip

---

### Phase 7: Run Build and Tests

**Objective:** Verify the dependency update doesn't break the project.

**Reference:** See `guides/build-test-verification.md` for detailed verification steps.

**Steps:**

1. **Detect Test Command:**
   - Check package.json scripts.test
   - Check Makefile for test target
   - Check CI configuration
   - Use ecosystem defaults

2. **Run Tests:**
   ```bash
   # Set timeout (10 minutes)
   timeout 600 {test_command}
   ```

3. **Capture Output:**
   - Save full test output to temporary file
   - Parse for pass/fail status
   - Extract error messages if failed

4. **Evaluate Results:**
   - **Tests Pass:** Proceed to Phase 9 (Create PR)
   - **Tests Fail:** Proceed to Phase 8 (Analyze Failures)

**Test Command Detection:**

The skill automatically detects the appropriate test command based on:
- Package manager configuration files
- CI/CD configuration
- Common conventions for the detected ecosystem

---

### Phase 8: Intelligent Auto-Fix (Migration Guide Discovery & Code Modification)

**Objective:** Automatically discover migration guides and apply code fixes when tests fail.

**Reference:**
- See `guides/migration-guide-discovery.md` for discovery strategies
- See `guides/code-modification.md` for fix implementation

**Approach:**
- Discover migration guides from multiple sources
- Parse guides for breaking changes and patterns
- Apply intelligent code modifications
- Retry tests up to 3 times with different fix strategies
- Create draft PR if fixes fail after max attempts

---

#### Phase 8a: Discover Migration Guides

**Steps:**

1. **Trigger Discovery:**
   ```bash
   # When tests fail, initiate migration guide discovery
   discover_migration_guide "$PACKAGE_NAME" "$OLD_VERSION" "$NEW_VERSION" "$ECOSYSTEM"
   ```

2. **4-Tier Search Strategy:**
   
   **Tier 1: GitHub Repository**
   - Check CHANGELOG.md, UPGRADING.md, MIGRATION.md
   - Check docs/migration/ directory
   - Fetch GitHub release notes
   ```bash
   search_github_migration_docs "$PACKAGE_NAME" "$OLD_VERSION" "$NEW_VERSION"
   ```
   
   **Tier 2: Package Registry**
   - Check package metadata and README
   - Look for project URLs and changelog
   - Search registry-specific documentation
   ```bash
   search_package_registry "$PACKAGE_NAME" "$ECOSYSTEM"
   ```
   
   **Tier 3: Official Documentation**
   - Search docs.{package}.com
   - Search {package}.readthedocs.io
   - Search official website
   ```bash
   search_official_docs "$PACKAGE_NAME" "$OLD_VERSION" "$NEW_VERSION"
   ```
   
   **Tier 4: Community Resources**
   - Search Stack Overflow
   - Search GitHub Issues
   - Document community resources for manual review
   ```bash
   search_community_resources "$PACKAGE_NAME" "$OLD_VERSION" "$NEW_VERSION"
   ```

3. **Handle Discovery Results:**
   - If guide found: Proceed to Phase 8b
   - If no guide found: Use fallback strategies (analyze test failures, search codebase)
   - Document all discovered resources

---

#### Phase 8b: Parse Migration Guides

**Steps:**

1. **Extract Breaking Changes:**
   ```bash
   extract_breaking_changes "migration-guide.md" > "breaking-changes.txt"
   ```

2. **Extract Code Patterns:**
   ```bash
   extract_migration_patterns "migration-guide.md" > "migration-patterns.json"
   ```

3. **Categorize Changes:**
   - Import/module changes
   - API signature changes
   - Configuration changes
   - Behavior changes
   ```bash
   categorize_changes "breaking-changes.txt" > "change-categories.json"
   ```

4. **Prepare for Code Modification:**
   ```bash
   prepare_for_code_modification "$PACKAGE_NAME"
   ```

---

#### Phase 8c: Apply Code Modifications

**Steps:**

1. **Analyze Test Failures:**
   ```bash
   analyze_test_failures_for_fixes "test-output.log" "$PACKAGE_NAME"
   ```

2. **Find Code Locations:**
   ```bash
   find_code_to_modify "$PACKAGE_NAME" "$FAILURE_TYPE"
   ```

3. **Apply Fixes by Type:**

   **Import Fixes:**
   ```bash
   fix_import_errors "$PACKAGE_NAME"
   ```
   
   **API Signature Fixes:**
   ```bash
   fix_api_signature_errors "$PACKAGE_NAME"
   ```
   
   **Configuration Fixes:**
   ```bash
   fix_config_errors "$PACKAGE_NAME"
   ```
   
   **Behavior Changes:**
   ```bash
   fix_behavior_changes "$PACKAGE_NAME"
   # Note: Behavior changes often require manual review
   ```

4. **Verify Syntax:**
   ```bash
   verify_syntax "$MODIFIED_FILE"
   ```

5. **Commit Changes:**
   ```bash
   git add -A
   git commit -m "Attempt 1: Fix $FAILURE_TYPE for $PACKAGE_NAME"
   ```

---

#### Phase 8d: Retry Build and Tests

**Steps:**

1. **Run Tests Again:**
   ```bash
   verify_fixes_with_tests "$TEST_COMMAND"
   ```

2. **Evaluate Results:**
   - **Tests Pass:** Proceed to Phase 9 (Create PR)
   - **Tests Still Fail:** Check progress and retry

3. **Iterative Fix Strategy (Max 3 Attempts):**
   ```bash
   apply_fixes_with_retry "$PACKAGE_NAME" "$TEST_COMMAND"
   ```
   
   **For each attempt:**
   - Analyze current failures
   - Apply different fix strategy
   - Commit changes
   - Run tests
   - Compare results
   
   **Progressive Strategy:**
   - Attempt 1: Import fixes (safest)
   - Attempt 2: Configuration fixes
   - Attempt 3: API signature fixes (more complex)

4. **Check Progress:**
   ```bash
   compare_test_results "test-output-before.log" "test-output-after.log"
   ```
   - If fewer failures: Continue with next attempt
   - If no progress: Try different strategy
   - If more failures: Rollback and try alternative approach

5. **After Max Attempts:**
   - If tests pass: Proceed to Phase 9 (Create PR)
   - If tests still fail: Proceed to Phase 10 (Create Draft PR)

---

#### Phase 8e: Rollback on Failure

**Steps:**

1. **Detect Failure Conditions:**
   - No progress after 3 attempts
   - Syntax errors introduced
   - More test failures than before

2. **Rollback Changes:**
   ```bash
   rollback_changes "No progress after 3 attempts"
   git reset --hard $START_COMMIT
   ```

3. **Preserve Diagnostic Information (with Sanitization):**
   - Save all attempted fixes
   - Save test outputs (RAW and SANITIZED versions)
   - Save discovered migration guides
   - Document what was tried
   - **Create sanitized versions of all diagnostic files**

4. **Proceed to Phase 10:**
   - Create draft PR with sanitized diagnostics
   - Include all discovered resources
   - Document attempted fixes
   - Provide recommendations for manual review
   - **Ensure all content is sanitized**

---

**Error Handling:**
- If migration guide discovery fails: Use fallback strategies
- If code modification introduces errors: Rollback that change
- If no progress after 3 attempts: Create draft PR
- If tests timeout: Treat as failure, document in draft PR

**Success Criteria:**
- Tests pass after code modifications
- No syntax errors introduced
- Changes are minimal and targeted
- All modifications are documented

---

### Phase 9: Create Pull Request

**Objective:** Create a PR for the successful dependency update.

**Reference:** See `guides/pr-creation.md` for PR formatting details.

**Steps:**

1. **⚠️ CRITICAL: Sanitize All Content (MANDATORY)**
   ```bash
   # Sanitize all content before creating PR
   sanitize_pr_content "pr-description-raw.md" > "pr-description.md"
   sanitize_test_output "test-output.log" > "test-output-sanitized.log"
   sanitize_build_output "build-output.log" > "build-output-sanitized.log"
   ```
   
   **Security Requirements:**
   - Remove all absolute paths (convert to relative)
   - Redact all credentials, tokens, API keys
   - Remove private IP addresses and internal hostnames
   - Redact environment variable values
   - Sanitize URLs with embedded credentials
   - Remove internal build server information
   
   **Reference:** See `guides/security-sanitization.md` for complete sanitization guide.

2. **Complete Security Checklist:**
   - Review `security-checklist.md`
   - Verify all sensitive information is redacted
   - Manually review all PR content
   - Confirm content is safe for public viewing

3. **Push Branch:**
   ```bash
   # Push to selected remote from Phase 1.5
   git push ${SELECTED_REMOTE} dependabot-autofix/{package-name}-{timestamp}
   ```

4. **Generate PR Description:**
   - Use template from `pr-template.md`
   - Include all fixed alerts
   - List version changes
   - Include verification results (SANITIZED ONLY)
   - Add migration notes if applicable
   - **Use ONLY sanitized files for PR content**

5. **Create PR:**
   ```bash
   # Create PR on selected remote's repository
   gh pr create \
     --repo ${SELECTED_REMOTE_OWNER_REPO} \
     --title "[Security] Fix Dependabot alerts for {package-name}" \
     --body "$(cat pr-description.md)" \
     --label "security,dependencies,automated" \
     --base main
   ```

6. **Record PR Details:**
   - PR number
   - PR URL
   - Status (open)

7. **Proceed to Phase 11** (Next Alert or Complete)

**Success Criteria:**
- ✅ All content sanitized before PR creation
- ✅ Security checklist completed
- ✅ No sensitive information in PR
- ✅ PR created successfully

---

### Phase 10: Create Draft PR (For Failed Fixes)

**Objective:** Create a draft PR when tests fail, documenting the issue.

**Steps:**

1. **⚠️ CRITICAL: Sanitize All Diagnostic Content (MANDATORY)**
   ```bash
   # Sanitize ALL diagnostic information before creating draft PR
   sanitize_pr_content "draft-pr-description-raw.md" > "draft-pr-description.md"
   sanitize_test_output "test-output.log" > "test-output-sanitized.log"
   sanitize_test_output "test-failure-details.txt" > "test-failure-details-sanitized.txt"
   sanitize_test_output "failure-analysis.txt" > "failure-analysis-sanitized.txt"
   sanitize_build_output "build-output.log" > "build-output-sanitized.log"
   ```
   
   **Security Requirements:**
   - Remove all absolute paths (convert to relative)
   - Redact all credentials, tokens, API keys
   - Remove private IP addresses and internal hostnames
   - Redact environment variable values
   - Sanitize error messages and stack traces
   - Remove internal build/test server information
   - Sanitize all code modification attempts
   
   **Reference:** See `guides/security-sanitization.md` for complete sanitization guide.

2. **Complete Security Checklist:**
   - Review `security-checklist.md`
   - Verify all diagnostic information is sanitized
   - Manually review all draft PR content
   - Confirm content is safe for public viewing
   - **Extra scrutiny for failed tests** (often contain more sensitive info)

3. **Push Branch:**
   ```bash
   # Push to selected remote from Phase 1.5
   git push ${SELECTED_REMOTE} dependabot-autofix/{package-name}-{timestamp}
   ```

4. **Generate Draft PR Description:**
   - Mark as requiring manual review
   - Include attempted changes
   - Document test failures (SANITIZED ONLY)
   - Include diagnostic output (SANITIZED ONLY)
   - Provide recommended next steps
   - Link to migration resources if found
   - **Use ONLY sanitized files for PR content**
   - Add note: "All diagnostic information has been sanitized"

5. **Create Draft PR:**
   ```bash
   # Create draft PR on selected remote's repository
   gh pr create \
     --repo ${SELECTED_REMOTE_OWNER_REPO} \
     --title "[Security] Fix Dependabot alerts for {package-name} (DRAFT - Manual Review Required)" \
     --body "$(cat draft-pr-description.md)" \
     --label "security,dependencies,automated,needs-work" \
     --draft \
     --base main
   ```

6. **Record PR Details:**
   - PR number
   - PR URL
   - Status (draft)

7. **Proceed to Phase 11** (Next Alert or Complete)

**Note on Remote Selection:**
- All PR operations use the remote selected in Phase 1.5
- Branch is pushed to the selected remote
- PR is created on the selected remote's repository
- Ensures consistency across the entire workflow

**Success Criteria:**
- ✅ All diagnostic content sanitized before PR creation
- ✅ Security checklist completed
- ✅ No sensitive information in draft PR
- ✅ Draft PR created successfully with clear manual review instructions

---

### Phase 11: Cleanup and Next Alert

**Objective:** Clean up current work and move to next alert or complete.

**Steps:**

1. **Return to Original Branch:**
   ```bash
   git checkout {original_branch}
   ```

2. **Update Progress Tracking:**
   - Mark current library as processed
   - Record PR created (or draft)
   - Update success/failure counts

3. **Check for More Alerts:**
   - If more libraries in selection: Return to Phase 5
   - If all processed: Proceed to Phase 13 (Generate Report)

---

### Phase 13: Generate Final Report

**Objective:** Provide comprehensive summary of all fixes attempted.

**Report Contents:**

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

2. library-b - PR #124
   - Fixed 1 alert (high)
   - Updated from 2.0.0 to 2.1.0
   - All tests passing
   - URL: https://github.com/owner/repo/pull/124

Failed Fixes (Draft PRs Created):
1. library-c - PR #125 (DRAFT)
   - Attempted to fix 2 alerts (medium)
   - Updated from 3.1.0 to 3.2.0
   - Tests failed - manual review required
   - URL: https://github.com/owner/repo/pull/125

Next Steps:
1. Review and merge PR #123 (library-a)
2. Review and merge PR #124 (library-b)
3. Manually fix and complete PR #125 (library-c)

Statistics:
- Success Rate: 67% (2/3 libraries)
- Alerts Fixed: 4/6 (67%)
- Time Taken: X minutes
```

---

## State Management

The skill maintains state throughout execution:

```json
{
  "session_id": "uuid",
  "started_at": "timestamp",
  "alert_remote": "upstream",
  "alert_remote_url": "https://github.com/org/repo.git",
  "alert_repository": "org/repo",
  "push_remote": "origin",
  "push_remote_url": "https://github.com/user/repo.git",
  "push_repository": "user/repo",
  "push_owner": "user",
  "fork_workflow": true,
  "original_branch": "main",
  "alerts_selected": [
    {
      "library": "example-library",
      "alerts": [1, 2, 3],
      "status": "processing|success|failed|skipped",
      "pr_number": 123,
      "pr_url": "https://..."
    }
  ],
  "current_library": "example-library",
  "prs_created": [],
  "prs_draft": []
}
```

**State Variables Explained:**
- `alert_remote`: Remote name where Dependabot alerts are fetched from
- `alert_repository`: Owner/repo for alert fetching and PR target
- `push_remote`: Remote name where branches are pushed to
- `push_repository`: Owner/repo for push operations
- `push_owner`: Owner name for fork workflow PR head reference
- `fork_workflow`: Boolean indicating if using fork workflow (different remotes)

## Error Handling

**Graceful Degradation:**
- If GitHub API fails: Retry with exponential backoff
- If package manager fails: Log error, create draft PR
- If tests timeout: Treat as failure, create draft PR
- If git operations fail: Rollback changes, report error

**Rollback Strategy:**
- Always return to original branch
- Delete feature branch if PR creation fails
- Preserve work in draft PR if possible
- Never leave repository in broken state

## Usage Examples

**Example 1: Fix all critical alerts**
```
User: "Fix all critical Dependabot alerts"
Skill: [Fetches alerts, filters by critical severity, processes each]
```

**Example 2: Fix specific library**
```
User: "Fix Dependabot alerts for library-name"
Skill: [Fetches alerts, filters for library-name, processes]
```

**Example 3: Fix top 5 alerts**
```
User: "Fix the top 5 most severe Dependabot alerts"
Skill: [Fetches alerts, sorts by severity, takes top 5, processes]
```

## Advanced Capabilities

This implementation includes:
- ✅ Alert fetching and filtering
- ✅ User selection with multiple options
- ✅ Dependency updates for any Dependabot-supported ecosystem
- ✅ Build and test verification
- ✅ Automatic migration guide discovery (4-tier strategy)
- ✅ Automatic code modification for breaking changes
- ✅ Retry logic with iterative fixes (up to 3 attempts)
- ✅ Intelligent pattern matching and code analysis
- ✅ Rollback mechanisms for failed fixes
- ✅ PR creation for successful fixes
- ✅ Draft PR creation with diagnostics for failed fixes
- ✅ Progress tracking and reporting

Future enhancements planned:
- ❌ Configuration file management
- ❌ Parallel processing of multiple alerts
- ❌ Machine learning-based fix suggestions
- ❌ Integration with external code analysis tools

## Testing the Skill

To test this skill:

1. **Setup Test Repository:**
   - Create or use a repository with Dependabot alerts
   - Ensure you have write access
   - Verify tests are working

2. **Run the Skill:**
   - Activate Bob in the repository
   - Say: "Use the Dependabot Auto-Fix skill to fix alerts"
   - Follow the prompts

3. **Verify Results:**
   - Check created PRs on GitHub
   - Verify tests pass in PRs
   - Review PR descriptions for completeness

4. **Test Edge Cases:**
   - Repository with no alerts
   - Repository with failing tests
   - Repository with multiple package managers
   - Repository with transitive dependencies

## Support and Troubleshooting

**Common Issues:**

1. **"gh not authenticated"**
   - Run: `gh auth login`
   - Follow authentication prompts

2. **"No test command found"**
   - Manually specify test command when prompted
   - Or add test script to package.json

3. **"Tests failing after update"**
   - Review draft PR for diagnostics
   - Check migration guide (if linked)
   - Manually apply necessary fixes

4. **"PR creation failed"**
   - Check GitHub permissions
   - Verify branch was pushed
   - Check for branch protection rules

## Contributing

To enhance this skill:
1. Improve test failure analysis
2. Enhance migration guide discovery
3. Improve automatic code fix patterns
4. Enhance error messages
5. Add more intelligent retry strategies

---

*Version: 2.0.0*
*Last Updated: 2026-05-26*