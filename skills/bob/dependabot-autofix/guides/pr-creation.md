# Pull Request Creation Guide

This guide provides detailed instructions for creating pull requests for Dependabot security fixes, including both successful fixes and draft PRs for failed attempts.

## Overview

After successfully updating a dependency and verifying tests pass, create a pull request to merge the changes. If tests fail, create a draft PR with diagnostic information for manual review.

**Important:** All PR operations use the TWO GitHub remotes selected in Phase 1.5:
1. **Push Remote**: Where branches are pushed to (typically your fork in fork workflows)
2. **Alert/PR Target Remote**: Where PRs are created to (typically upstream in fork workflows)

This two-remote approach supports the common fork workflow where you push to your fork and create PRs to the upstream repository.

## ⚠️ CRITICAL: Security Sanitization

**BEFORE creating any PR, ALL content MUST be sanitized to prevent sensitive information leakage.**

### Mandatory Security Steps

1. **Read the Security Guide**: Review [security-sanitization.md](./security-sanitization.md)
2. **Apply Sanitization**: Sanitize all PR content (description, diagnostics, test output, error messages)
3. **Complete Checklist**: Use [security-checklist.md](../security-checklist.md) to verify
4. **Manual Review**: Manually review all content for sensitive information
5. **Only Then Create PR**: Create PR only after sanitization is complete

### What Must Be Sanitized

- ✅ **Test output** - Remove credentials, absolute paths, internal IPs
- ✅ **Error messages** - Redact sensitive values, connection strings
- ✅ **Stack traces** - Convert absolute paths to relative paths
- ✅ **Build logs** - Remove internal server info, credentials
- ✅ **Configuration values** - Redact all config values
- ✅ **Environment variables** - Redact values (keep names)
- ✅ **URLs** - Remove embedded credentials and tokens
- ✅ **File paths** - Use relative paths only

### Quick Sanitization Reference

```bash
# Before creating PR, sanitize all content:
sanitize_content() {
    local content="$1"
    
    # Redact absolute paths
    content=$(echo "$content" | sed -E 's|/Users/[^/]+|/[USER_HOME]|g')
    content=$(echo "$content" | sed -E 's|/home/[^/]+|/[USER_HOME]|g')
    
    # Redact tokens/keys
    content=$(echo "$content" | sed -E 's/[A-Za-z0-9_-]{32,}/[REDACTED_TOKEN]/g')
    
    # Redact URLs with credentials
    content=$(echo "$content" | sed -E 's|://[^:]+:[^@]+@|://[REDACTED]@|g')
    
    # Redact private IPs
    content=$(echo "$content" | sed -E 's/192\.168\.[0-9.]+/[PRIVATE_IP]/g')
    content=$(echo "$content" | sed -E 's/10\.[0-9.]+/[PRIVATE_IP]/g')
    
    echo "$content"
}

# Apply to all PR content before creating PR
PR_DESCRIPTION=$(sanitize_content "$PR_DESCRIPTION")
TEST_OUTPUT=$(sanitize_content "$TEST_OUTPUT")
ERROR_MESSAGES=$(sanitize_content "$ERROR_MESSAGES")
```

**See [security-sanitization.md](./security-sanitization.md) for complete sanitization patterns and examples.**

## PR Strategy

### One PR Per Library

- Group all alerts for the same library into a single PR
- Single PR addresses all CVEs for that library
- Cleaner review process
- Easier to track and merge

**Example:**
- If lodash has 3 alerts, create 1 PR fixing all 3
- If requests has 1 alert, create 1 PR fixing that 1
- Total: 2 PRs for 4 alerts

### Branch Naming Convention

```
dependabot-autofix/{package-name}-{timestamp}
```

**Examples:**
```
dependabot-autofix/lodash-20260526
dependabot-autofix/requests-20260526143022
dependabot-autofix/spring-core-20260526
```

**Implementation:**
```bash
TIMESTAMP=$(date +%Y%m%d%H%M%S)
BRANCH_NAME="dependabot-autofix/${PACKAGE_NAME}-${TIMESTAMP}"
git checkout -b "$BRANCH_NAME"
```

---

## Successful Fix PR

### Title Format

```
[Security] Fix Dependabot alerts for {package-name}
```

**Examples:**
- `[Security] Fix Dependabot alerts for lodash`
- `[Security] Fix Dependabot alerts for requests`
- `[Security] Fix Dependabot alerts for spring-core`

### PR Description Template

```markdown
## Summary
This PR fixes {N} Dependabot security alert(s) for `{package-name}`.

## Alerts Fixed
{For each alert:}
- **Alert #{number}**: {CVE-ID} - {severity} - {summary}
  - Vulnerable version: {old-version}
  - Fixed version: {new-version}
  - CVSS Score: {score}

## Changes Made
- Updated `{package-name}` from `{old-version}` to `{new-version}`

## Verification
- ✅ Build successful
- ✅ All unit tests passing ({N} tests)

## Verification
- ✅ Build successful
- ✅ All unit tests passing ({N} tests)

## Code Modifications
{If code modifications were applied:}
**Automatic fixes applied:** {N} fix attempt(s)

### Changes Made:
- {Description of import fixes, if any}
- {Description of API signature fixes, if any}
- {Description of configuration fixes, if any}

### Migration Guide Used:
- Source: {GitHub/Package Registry/Official Docs/Community}
- URL: {migration-guide-url}

{If no code modifications:}
No code modifications required - dependency update was compatible.

## Additional Notes
{Any relevant notes, warnings, or follow-up items}

---
*This PR was automatically created by the Dependabot Auto-Fix skill.*
```

### Example PR Description

```markdown
## Summary
This PR fixes 3 Dependabot security alerts for `lodash`.

## Alerts Fixed

- **Alert #1**: CVE-2021-23337 - high - Command Injection in lodash
  - Vulnerable version: 4.17.15
  - Fixed version: 4.17.21
  - CVSS Score: 7.2

- **Alert #2**: CVE-2020-28500 - high - Regular Expression Denial of Service (ReDoS) in lodash
  - Vulnerable version: 4.17.15
  - Fixed version: 4.17.21
  - CVSS Score: 7.5

- **Alert #3**: CVE-2019-10744 - critical - Prototype Pollution in lodash
  - Vulnerable version: 4.17.15
  - Fixed version: 4.17.21
  - CVSS Score: 9.1

## Changes Made
- Updated `lodash` from `4.17.15` to `4.17.21`

## Verification
- ✅ Build successful
- ✅ All unit tests passing (127 tests)

## Code Modifications
No code modifications required - dependency update was compatible.

## Additional Notes
This is a patch update with no breaking changes. All tests pass without modification.

---
*This PR was automatically created by the Dependabot Auto-Fix skill.*
```

---

## Draft PR for Failed Fixes

### Title Format

```
[Security] Fix Dependabot alerts for {package-name} (DRAFT - Manual Review Required)
```

**Examples:**
- `[Security] Fix Dependabot alerts for spring-core (DRAFT - Manual Review Required)`
- `[Security] Fix Dependabot alerts for django (DRAFT - Manual Review Required)`

### Draft PR Description Template

```markdown
## ⚠️ Draft PR - Manual Review Required

This PR was automatically created but requires manual intervention to complete.

## Alerts to Fix

{For each alert:}
- **Alert #{number}**: {CVE-ID} - {severity} - {summary}
  - Vulnerable version: {old-version}
  - Fixed version: {new-version}
  - CVSS Score: {score}

## Attempted Changes
- Updated `{package-name}` from `{old-version}` to `{new-version}`

## Issues Encountered

### Test Failures
{List failing tests:}
- `{test-name}`: {brief error description}

### Error Summary
{Categorize errors:}
- Import/Module Errors: {count}
- API Signature Changes: {count}
- Behavior Changes: {count}
- Other: {count}

## Diagnostic Information

<details>
<summary>Test Output</summary>

```
{Full test output - first 100 lines}
```
</details>

<details>
<summary>Build Output (if applicable)</summary>

```
{Full build output}
```
</details>

<details>
<summary>Failure Analysis</summary>

{Categorized failure analysis from build-test-verification.md}

**Failure Category**: {Import Error | API Change | Behavior Change | Configuration Error}

**Affected Areas**:
- {List of affected files/modules}

**Suspected Breaking Changes**:
- {List of suspected breaking changes based on error messages}

</details>

## Auto-Fix Attempts

**Attempts made:** {N} of 3

### Attempt 1: {Fix Type}
- Changes: {Description of changes attempted}
- Result: {Pass/Fail - X failures remaining}

### Attempt 2: {Fix Type}
- Changes: {Description of changes attempted}
- Result: {Pass/Fail - X failures remaining}

### Attempt 3: {Fix Type}
- Changes: {Description of changes attempted}
- Result: {Pass/Fail - X failures remaining}

**Outcome:** All automatic fix attempts exhausted. Manual intervention required.

<details>
<summary>Applied Code Modifications</summary>

{List of files modified and changes made during auto-fix attempts}

</details>

## Recommended Next Steps

1. Review the test failures above
2. Review the auto-fix attempts and why they failed
3. Check the migration guide: {URL if available, or "Not found"}
4. Manually apply necessary code changes to address:
   - {Specific issue 1}
   - {Specific issue 2}
5. Run tests locally to verify: `{test-command}`
6. Push additional commits to this branch
7. Mark PR as ready for review once tests pass

## Migration Resources

{If found:}
- Migration guide: {URL}
- Changelog: {URL}
- Release notes: {URL}
- Related issues: {URLs}

{If not found:}
- ⚠️ No migration guide found
- Check package documentation: {package-homepage-url}
- Search for breaking changes in release notes

## Manual Fix Hints

Based on the error analysis, consider:

{For Import Errors:}
- Check if imports have been renamed or moved
- Verify all required packages are installed
- Update import statements

{For API Changes:}
- Review function signatures in the new version
- Update function calls to match new signatures
- Check for deprecated methods

{For Behavior Changes:}
- Review test expectations
- Check if return values have changed
- Verify side effects match expectations

---
*This draft PR was automatically created by the Dependabot Auto-Fix skill.*
*Tests failed after {N} attempts. Manual intervention required.*
```

### Example Draft PR Description

```markdown
## ⚠️ Draft PR - Manual Review Required

This PR was automatically created but requires manual intervention to complete.

## Alerts to Fix

- **Alert #5**: CVE-2023-12345 - medium - Security vulnerability in spring-core
  - Vulnerable version: 5.3.15
  - Fixed version: 5.3.20
  - CVSS Score: 6.5

- **Alert #6**: CVE-2023-12346 - medium - Another vulnerability in spring-core
  - Vulnerable version: 5.3.15
  - Fixed version: 5.3.20
  - CVSS Score: 6.2

## Attempted Changes
- Updated `spring-core` from `5.3.15` to `5.3.20`

## Issues Encountered

### Test Failures
- `ApplicationContextTest.testBeanCreation`: BeanCreationException
- `WebConfigTest.testMvcConfig`: NoSuchMethodError
- `SecurityConfigTest.testSecurityChain`: ClassNotFoundException

### Error Summary
- Import/Module Errors: 1
- API Signature Changes: 2
- Behavior Changes: 0
- Other: 0

## Diagnostic Information

<details>
<summary>Test Output</summary>

```
[ERROR] Tests run: 45, Failures: 3, Errors: 0, Skipped: 0

[ERROR] testBeanCreation(com.example.ApplicationContextTest)
  org.springframework.beans.factory.BeanCreationException: 
  Error creating bean with name 'dataSource': 
  Invocation of init method failed; nested exception is 
  java.lang.NoSuchMethodError: 
  org.springframework.core.env.Environment.getProperty(Ljava/lang/String;)Ljava/lang/String;

[ERROR] testMvcConfig(com.example.WebConfigTest)
  java.lang.NoSuchMethodError: 
  org.springframework.web.servlet.config.annotation.WebMvcConfigurer.addResourceHandlers
  (Lorg/springframework/web/servlet/config/annotation/ResourceHandlerRegistry;)V

[ERROR] testSecurityChain(com.example.SecurityConfigTest)
  java.lang.ClassNotFoundException: 
  org.springframework.security.config.annotation.web.builders.HttpSecurity$RequestMatcherConfigurer
```
</details>

<details>
<summary>Failure Analysis</summary>

**Failure Category**: API Signature Changes

**Affected Areas**:
- ApplicationContext configuration
- WebMvc configuration
- Security configuration

**Suspected Breaking Changes**:
- `Environment.getProperty()` method signature changed
- `WebMvcConfigurer.addResourceHandlers()` method signature changed
- Security configuration classes reorganized

</details>

## Recommended Next Steps

1. Review the test failures above
2. Check the migration guide: https://github.com/spring-projects/spring-framework/wiki/Upgrading-to-Spring-Framework-5.x
3. Manually apply necessary code changes to address:
   - Update Environment.getProperty() calls
   - Update WebMvcConfigurer implementation
   - Update Security configuration imports
4. Run tests locally to verify: `mvn test`
5. Push additional commits to this branch
6. Mark PR as ready for review once tests pass

## Migration Resources

- Migration guide: https://github.com/spring-projects/spring-framework/wiki/Upgrading-to-Spring-Framework-5.x
- Changelog: https://github.com/spring-projects/spring-framework/releases/tag/v5.3.20
- Release notes: https://spring.io/blog/2022/03/31/spring-framework-5-3-20-available-now

## Auto-Fix Attempts

**Attempts made:** 3 of 3

### Attempt 1: Import Fixes
- Changes: Updated import paths for reorganized modules
- Result: Failed - 2 failures remaining

### Attempt 2: Configuration Fixes
- Changes: Updated configuration format for new version
- Result: Failed - 2 failures remaining (no progress)

### Attempt 3: API Signature Fixes
- Changes: Updated method calls to match new signatures
- Result: Failed - 3 failures (introduced new issues, rolled back)

**Outcome:** All automatic fix attempts exhausted. Manual intervention required.

<details>
<summary>Applied Code Modifications</summary>

Files modified during auto-fix attempts:
- `src/config/ApplicationContext.java` - Attempted to fix bean creation
- `src/config/WebConfig.java` - Attempted to fix MVC configuration
- `src/config/SecurityConfig.java` - Attempted to fix security imports

All changes were rolled back after final attempt failed.

</details>

## Manual Fix Hints

Based on the error analysis, consider:

**For API Changes:**
- Review function signatures in Spring Framework 5.3.20
- Update method calls to match new signatures
- Check Spring documentation for deprecated methods

**Specific Issues:**
1. `Environment.getProperty()` - Check if return type or parameters changed
2. `WebMvcConfigurer.addResourceHandlers()` - Verify interface method signature
3. Security classes - Check if packages were reorganized

---
*This draft PR was automatically created by the Dependabot Auto-Fix skill.*
*Tests failed after 3 automatic fix attempts. Manual intervention required.*
```

---

## PR Creation Commands

### Using GitHub CLI

**Create Branch:**
```bash
PACKAGE_NAME="lodash"
TIMESTAMP=$(date +%Y%m%d)
BRANCH_NAME="dependabot-autofix/${PACKAGE_NAME}-${TIMESTAMP}"

git checkout -b "$BRANCH_NAME"
```

**Commit Changes:**
```bash
git add .
git commit -m "[Security] Fix Dependabot alerts for ${PACKAGE_NAME}"
```

**Push Branch:**
```bash
# Push to push remote from Phase 1.5
git push ${PUSH_REMOTE} "$BRANCH_NAME"
```

**Create PR (Success):**

For **Direct Push Workflow** (same remote):
```bash
# Create PR on alert remote's repository
gh pr create \
  --repo ${ALERT_REPOSITORY} \
  --title "[Security] Fix Dependabot alerts for ${PACKAGE_NAME}" \
  --body "$(cat pr-description.md)" \
  --label "security,dependencies,automated" \
  --base main
```

For **Fork Workflow** (different remotes):
```bash
# Create PR from fork to upstream
gh pr create \
  --repo ${ALERT_REPOSITORY} \
  --head ${PUSH_OWNER}:${BRANCH_NAME} \
  --title "[Security] Fix Dependabot alerts for ${PACKAGE_NAME}" \
  --body "$(cat pr-description.md)" \
  --label "security,dependencies,automated" \
  --base main
```

**Create Draft PR (Failure):**

For **Direct Push Workflow** (same remote):
```bash
# Create draft PR on alert remote's repository
gh pr create \
  --repo ${ALERT_REPOSITORY} \
  --title "[Security] Fix Dependabot alerts for ${PACKAGE_NAME} (DRAFT - Manual Review Required)" \
  --body "$(cat draft-pr-description.md)" \
  --label "security,dependencies,automated,needs-work" \
  --draft \
  --base main
```

For **Fork Workflow** (different remotes):
```bash
# Create draft PR from fork to upstream
gh pr create \
  --repo ${ALERT_REPOSITORY} \
  --head ${PUSH_OWNER}:${BRANCH_NAME} \
  --title "[Security] Fix Dependabot alerts for ${PACKAGE_NAME} (DRAFT - Manual Review Required)" \
  --body "$(cat draft-pr-description.md)" \
  --label "security,dependencies,automated,needs-work" \
  --draft \
  --base main
```

**Note:**
- `PUSH_REMOTE` and `ALERT_REPOSITORY` are set during Phase 1.5 (Remote Selection)
- `PUSH_OWNER` is extracted from the push repository for fork workflow
- `--head` parameter is only needed for fork workflow to specify `user:branch`

### Complete PR Creation Workflow

```bash
#!/bin/bash

create_pr() {
    local package_name="$1"
    local old_version="$2"
    local new_version="$3"
    local alerts="$4"  # JSON array of alerts
    local tests_passed="$5"  # true/false
    
    # PUSH_REMOTE, ALERT_REPOSITORY, PUSH_OWNER, and FORK_WORKFLOW should be set from Phase 1.5
    if [ -z "$PUSH_REMOTE" ] || [ -z "$ALERT_REPOSITORY" ]; then
        echo "Error: Remotes not selected. Run Phase 1.5 first."
        exit 1
    fi
    
    # Generate branch name
    TIMESTAMP=$(date +%Y%m%d%H%M%S)
    BRANCH_NAME="dependabot-autofix/${package_name}-${TIMESTAMP}"
    
    # Create and checkout branch
    git checkout -b "$BRANCH_NAME"
    
    # Commit changes
    git add .
    git commit -m "[Security] Fix Dependabot alerts for ${package_name}"
    
    # Push branch to push remote
    echo "Pushing to remote: $PUSH_REMOTE"
    git push "$PUSH_REMOTE" "$BRANCH_NAME"
    
    # Determine PR creation command based on workflow type
    if [ "$FORK_WORKFLOW" = "true" ]; then
        HEAD_REF="${PUSH_OWNER}:${BRANCH_NAME}"
        echo "Fork workflow: Creating PR from ${HEAD_REF} to ${ALERT_REPOSITORY}"
    else
        HEAD_REF=""
        echo "Direct push workflow: Creating PR on ${ALERT_REPOSITORY}"
    fi
    
    # Generate PR description
    if [ "$tests_passed" = "true" ]; then
        generate_success_pr_description "$package_name" "$old_version" "$new_version" "$alerts" > pr-description.md
        
        # Create PR on alert remote's repository
        echo "Creating PR on repository: $ALERT_REPOSITORY"
        if [ "$FORK_WORKFLOW" = "true" ]; then
            gh pr create \
              --repo "$ALERT_REPOSITORY" \
              --head "$HEAD_REF" \
              --title "[Security] Fix Dependabot alerts for ${package_name}" \
              --body "$(cat pr-description.md)" \
              --label "security,dependencies,automated" \
              --base main
        else
            gh pr create \
              --repo "$ALERT_REPOSITORY" \
              --title "[Security] Fix Dependabot alerts for ${package_name}" \
              --body "$(cat pr-description.md)" \
              --label "security,dependencies,automated" \
              --base main
        fi
    else
        generate_draft_pr_description "$package_name" "$old_version" "$new_version" "$alerts" > draft-pr-description.md
        
        # Create Draft PR on alert remote's repository
        echo "Creating draft PR on repository: $ALERT_REPOSITORY"
        if [ "$FORK_WORKFLOW" = "true" ]; then
            gh pr create \
              --repo "$ALERT_REPOSITORY" \
              --head "$HEAD_REF" \
              --title "[Security] Fix Dependabot alerts for ${package_name} (DRAFT - Manual Review Required)" \
              --body "$(cat draft-pr-description.md)" \
              --label "security,dependencies,automated,needs-work" \
              --draft \
              --base main
        else
            gh pr create \
              --repo "$ALERT_REPOSITORY" \
              --title "[Security] Fix Dependabot alerts for ${package_name} (DRAFT - Manual Review Required)" \
              --body "$(cat draft-pr-description.md)" \
              --label "security,dependencies,automated,needs-work" \
              --draft \
              --base main
        fi
    fi
    
    # Capture PR URL
    PR_URL=$(gh pr view --json url -q .url)
    echo "PR created: $PR_URL"
    
    # Return to original branch
    git checkout -
}
```

---

## PR Labels

### Recommended Labels

- `security` - Indicates security fix
- `dependencies` - Dependency update
- `automated` - Automatically created
- `needs-work` - For draft PRs requiring manual fixes

### Creating Labels (if they don't exist)

```bash
gh label create security --color "d73a4a" --description "Security vulnerability fix"
gh label create dependencies --color "0366d6" --description "Dependency updates"
gh label create automated --color "ededed" --description "Automatically created"
gh label create needs-work --color "fbca04" --description "Requires manual intervention"
```

---

## PR Metadata

### Linking to Dependabot Alerts

In the PR description, reference alert numbers:
```markdown
Fixes #1, #2, #3
```

This helps GitHub track which alerts are addressed.

### Assignees and Reviewers

**Auto-assign:**
```bash
gh pr create \
  --assignee @me \
  --reviewer team-security
```

**Or let team handle assignment based on CODEOWNERS**

---

## Post-PR Creation

### Record PR Information

```bash
# Store PR details for final report
PR_NUMBER=$(gh pr view --json number -q .number)
PR_URL=$(gh pr view --json url -q .url)
PR_STATUS=$(gh pr view --json state -q .state)

echo "PR #${PR_NUMBER}: ${PR_URL} (${PR_STATUS})" >> pr-summary.txt
```

### Cleanup

```bash
# Return to original branch
git checkout main

# Clean up temporary files
rm -f pr-description.md draft-pr-description.md test-output.log
```

---

## PR Description Generation Functions

### Success PR Description

```bash
generate_success_pr_description() {
    local package_name="$1"
    local old_version="$2"
    local new_version="$3"
    local alerts="$4"  # JSON array
    
    cat <<EOF
## Summary
This PR fixes $(echo "$alerts" | jq length) Dependabot security alert(s) for \`${package_name}\`.

## Alerts Fixed

$(echo "$alerts" | jq -r '.[] | "- **Alert #\(.number)**: \(.cve_id) - \(.severity) - \(.summary)\n  - Vulnerable version: \(.vulnerable_version)\n  - Fixed version: \(.patched_version)\n  - CVSS Score: \(.cvss_score)\n"')

## Changes Made
- Updated \`${package_name}\` from \`${old_version}\` to \`${new_version}\`

## Verification
- ✅ Build successful
- ✅ All unit tests passing

## Additional Notes
This is a security patch update. All tests pass without modification.

---
*This PR was automatically created by the Dependabot Auto-Fix skill.*
EOF
}
```

### Draft PR Description

```bash
generate_draft_pr_description() {
    local package_name="$1"
    local old_version="$2"
    local new_version="$3"
    local alerts="$4"  # JSON array
    
    cat <<EOF
## ⚠️ Draft PR - Manual Review Required

This PR was automatically created but requires manual intervention to complete.

## Alerts to Fix

$(echo "$alerts" | jq -r '.[] | "- **Alert #\(.number)**: \(.cve_id) - \(.severity) - \(.summary)\n  - Vulnerable version: \(.vulnerable_version)\n  - Fixed version: \(.patched_version)\n  - CVSS Score: \(.cvss_score)\n"')

## Attempted Changes
- Updated \`${package_name}\` from \`${old_version}\` to \`${new_version}\`

## Issues Encountered

### Test Failures
Tests failed after dependency update. See diagnostic information below.

## Diagnostic Information

<details>
<summary>Test Output</summary>

\`\`\`
$(cat test-output.log | head -100)
\`\`\`
</details>

<details>
<summary>Failure Analysis</summary>

$(cat failure-analysis.txt)

</details>

## Recommended Next Steps

1. Review the test failures above
2. Check the migration guide (if available)
3. Manually apply necessary code changes
4. Run tests locally to verify
5. Push additional commits to this branch
6. Mark PR as ready for review once tests pass

---
*This draft PR was automatically created by the Dependabot Auto-Fix skill.*
*Tests failed. Manual intervention required.*
EOF
}
```

---

## Best Practices

1. **Security First**
   - **ALWAYS sanitize content before creating PR**
   - Complete security checklist
   - Manual review of all content
   - Never include credentials, tokens, or internal paths

2. **Clear Titles**
   - Use consistent format
   - Include [Security] prefix
   - Mention package name

3. **Comprehensive Descriptions**
   - List all alerts fixed
   - Show version changes
   - Include verification results
   - **Ensure all content is sanitized**

4. **Proper Labels**
   - Always add security label
   - Add dependencies label
   - Add automated label

5. **Link to Alerts**
   - Reference alert numbers
   - Helps GitHub tracking

6. **Include Diagnostics (Draft PRs)**
   - **Sanitized test output only**
   - **Sanitized failure analysis**
   - Recommended fixes (no sensitive info)

7. **Clean Branch Names**
   - Use consistent naming
   - Include timestamp
   - Easy to identify

8. **Security Verification**
   - Use relative paths only
   - Redact all credentials
   - Remove internal network info
   - Sanitize error messages

---

## Troubleshooting

### Issue: PR Creation Fails

**Check:**
```bash
# Verify authentication
gh auth status

# Verify branch pushed
git branch -r | grep dependabot-autofix

# Check permissions
gh api repos/{owner}/{repo} --jq .permissions
```

### Issue: Cannot Push Branch

**Solutions:**
```bash
# Check remote
git remote -v

# Verify push remote exists
git remote get-url "$PUSH_REMOTE"

# Verify write access to push remote's repository
gh api "repos/$PUSH_REPOSITORY/collaborators/$(gh api user -q .login)/permission"

# Check authentication for push remote
gh auth status

# Force push if needed (with caution)
git push -f "$PUSH_REMOTE" "$BRANCH_NAME"
```

### Issue: Wrong Repository for PR

**Problem:** PR created on wrong repository (e.g., fork instead of upstream)

**Solution:**
```bash
# Verify selected remotes
echo "Alert remote: $ALERT_REMOTE ($ALERT_REPOSITORY)"
echo "Push remote: $PUSH_REMOTE ($PUSH_REPOSITORY)"
echo "Fork workflow: $FORK_WORKFLOW"

# If wrong remotes were selected, re-run Phase 1.5
# Or manually specify correct repository:
gh pr create --repo correct-owner/correct-repo --head user:branch ...
```

### Issue: Fork Workflow Not Detected

**Problem:** Using fork workflow but PR created without --head parameter

**Solution:**
```bash
# Verify fork workflow is detected
echo "Fork workflow: $FORK_WORKFLOW"

# If fork workflow not detected but should be:
if [ "$ALERT_REMOTE" != "$PUSH_REMOTE" ]; then
    FORK_WORKFLOW=true
fi

# Manually create PR with --head for fork workflow:
gh pr create \
  --repo "$ALERT_REPOSITORY" \
  --head "${PUSH_OWNER}:${BRANCH_NAME}" \
  --title "..." \
  --body "..."
```

### Issue: Authentication for Different Remotes

**Problem:** Different remotes may require different authentication

**Solutions:**
```bash
# Check current authentication
gh auth status

# Switch GitHub account if needed
gh auth switch

# Login to different account
gh auth login

# Verify access to alert remote's repository
gh api "repos/$ALERT_REPOSITORY"

# Verify access to push remote's repository (if different)
if [ "$FORK_WORKFLOW" = "true" ]; then
    gh api "repos/$PUSH_REPOSITORY"
fi
```

### Issue: Push Remote Not a Fork

**Problem:** Push remote is not a fork of alert remote

**Solution:**
```bash
# Check if push repository is a fork
PARENT=$(gh api "repos/$PUSH_REPOSITORY" --jq '.parent.full_name' 2>/dev/null || echo "")

if [ -n "$PARENT" ]; then
    echo "Push repository is a fork of: $PARENT"
    if [ "$PARENT" != "$ALERT_REPOSITORY" ]; then
        echo "⚠️  Warning: Fork parent doesn't match alert repository"
        echo "   Expected: $ALERT_REPOSITORY"
        echo "   Actual: $PARENT"
    fi
else
    echo "⚠️  Warning: Push repository is not a fork"
    echo "   This may cause issues with PR creation"
fi

# If not a fork, consider using direct push workflow instead
```

### Issue: PR Description Too Long

**Solutions:**
- Truncate test output
- Use collapsible sections
- Link to external logs
- Summarize instead of full output

---

## Summary

This guide covers:
- ✅ **Security sanitization (MANDATORY)**
- ✅ Security checklist verification
- ✅ **Two-remote selection for PR creation (alert and push)**
- ✅ **Fork workflow support with --head parameter**
- ✅ **Using push remote for branches, alert remote for PRs**
- ✅ PR strategy (one per library)
- ✅ Branch naming conventions
- ✅ Success PR format and content
- ✅ Draft PR format and content
- ✅ Code modification documentation in PRs
- ✅ Auto-fix attempt tracking in draft PRs
- ✅ Migration guide references
- ✅ PR creation commands with sanitization and fork workflow support
- ✅ Labels and metadata
- ✅ Description generation with sanitization
- ✅ Best practices including security
- ✅ **Handling fork workflows (origin → upstream)**
- ✅ **Handling direct push workflows (same remote)**
- ✅ **Authentication for different remotes**

**Critical Security Reminders:**
- ⚠️ **NEVER create a PR without sanitizing content first**
- ⚠️ **Complete the security checklist for every PR**
- ⚠️ **Manually review all content before creating PR**
- ⚠️ **When in doubt, redact**

**Fork Workflow Reminders:**
- ⚠️ **Always use PUSH_REMOTE for git push operations**
- ⚠️ **Always use ALERT_REPOSITORY for gh pr create --repo**
- ⚠️ **Use --head ${PUSH_OWNER}:${BRANCH} for fork workflow**
- ⚠️ **Verify fork workflow detection (FORK_WORKFLOW variable)**
- ⚠️ **Ensure authentication works for both remotes**
- ⚠️ **Verify push remote is a fork of alert remote (if applicable)**

**Next Steps:**
- **Complete security sanitization**
- **Verify with security checklist**
- **Verify correct remotes are selected**
- **Verify fork workflow is properly detected**
- Record PR details for final report
- Document all code modifications applied
- Include auto-fix attempt history in draft PRs
- Clean up temporary files (including unsanitized versions)
- Move to next alert or complete

---

*Part of Dependabot Auto-Fix Skill v2.0.0*