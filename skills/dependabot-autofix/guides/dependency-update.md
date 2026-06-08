# Dependency Update Guide

This guide provides a general-purpose approach for updating dependencies to fix Dependabot security alerts, working with any package ecosystem that Dependabot supports.

## Overview

The dependency update process is ecosystem-agnostic and involves:
1. Auto-detecting the package manager from project files
2. Identifying the dependency file(s) for the detected ecosystem
3. Determining the current and target versions
4. Updating the dependency file
5. Running the appropriate package manager update command
6. Verifying lock files are updated (if applicable)
7. Committing the changes

## General Update Strategy

### Step-by-Step Process

1. **Auto-Detect Ecosystem**
   - Scan project for package manager files
   - Identify ecosystem from alert data (`dependency.package.ecosystem`)
   - Verify by checking for dependency files

2. **Locate Dependency File**
   - Use `manifest_path` from alert
   - Verify file exists
   - Identify associated lock files

3. **Determine Versions**
   - Current: Parse from dependency file
   - Target: Use `first_patched_version` from alert
   - Fallback: Use latest stable if patched version unavailable

4. **Update Dependency**
   - Modify dependency file using appropriate method
   - Run package manager's update command
   - Verify lock file changes (if applicable)

5. **Commit Changes**
   - Stage dependency file and lock file(s)
   - Create descriptive commit message

## Update Implementation

### Ecosystem Detection

```bash
# Auto-detect package manager from project files
detect_package_manager() {
    if [ -f "package.json" ]; then
        if [ -f "pnpm-lock.yaml" ]; then
            echo "pnpm"
        elif [ -f "yarn.lock" ]; then
            echo "yarn"
        else
            echo "npm"
        fi
    elif [ -f "requirements.txt" ] || [ -f "Pipfile" ] || [ -f "pyproject.toml" ]; then
        echo "python"
    elif [ -f "pom.xml" ]; then
        echo "maven"
    elif [ -f "build.gradle" ] || [ -f "build.gradle.kts" ]; then
        echo "gradle"
    elif [ -f "Gemfile" ]; then
        echo "bundler"
    elif [ -f "composer.json" ]; then
        echo "composer"
    elif [ -f "go.mod" ]; then
        echo "go"
    elif [ -f "Cargo.toml" ]; then
        echo "cargo"
    elif [ -f "*.csproj" ]; then
        echo "nuget"
    else
        echo "unknown"
    fi
}
```

### Generic Update Workflow

```bash
#!/bin/bash

update_dependency() {
    local package_name="$1"
    local target_version="$2"
    local ecosystem="$3"
    
    echo "Updating $package_name to $target_version in $ecosystem ecosystem..."
    
    # Use ecosystem-specific update command
    case "$ecosystem" in
        npm|yarn|pnpm|python|maven|gradle|bundler|composer|go|cargo|nuget)
            run_update_command "$ecosystem" "$package_name" "$target_version"
            ;;
        *)
            echo "Error: Unsupported ecosystem: $ecosystem"
            return 1
            ;;
    esac
    
    # Verify changes
    verify_dependency_updated "$package_name" "$target_version"
    
    # Commit changes
    commit_dependency_update "$package_name" "$target_version"
}

run_update_command() {
    local ecosystem="$1"
    local package="$2"
    local version="$3"
    
    # Execute the appropriate update command for the ecosystem
    # Implementation varies by package manager
    echo "Running update command for $ecosystem..."
}
```

### Version Resolution

```bash
get_target_version() {
    local alert_data="$1"
    
    # Try to get patched version from alert
    local patched_version=$(echo "$alert_data" | jq -r '.security_vulnerability.first_patched_version.identifier')
    
    if [ -n "$patched_version" ] && [ "$patched_version" != "null" ]; then
        echo "$patched_version"
    else
        # Fallback to latest stable version
        get_latest_stable_version "$package_name" "$ecosystem"
    fi
}
```

### Verification Steps

After updating a dependency:

1. **Check Dependency File:**
   ```bash
   git diff {dependency-file}
   ```

2. **Check Lock File (if applicable):**
   ```bash
   git diff {lock-file}
   ```

3. **Verify Version Installed:**
   ```bash
   # Use ecosystem-specific command to verify
   verify_installed_version "$package_name"
   ```

4. **Check for Conflicts:**
   ```bash
   # Run ecosystem-specific conflict check
   check_dependency_conflicts
   ```

## Handling Transitive Dependencies

### Identifying Transitive Dependencies

```bash
is_transitive_dependency() {
    local package_name="$1"
    local ecosystem="$2"
    
    # Check if dependency is direct or transitive
    # Implementation varies by ecosystem
    # Returns: 0 if transitive, 1 if direct
    check_dependency_type "$package_name" "$ecosystem"
}
```

### Updating Transitive Dependencies

**IMPORTANT: Always follow this prioritization strategy when handling transitive dependencies.**

#### Option 1: Update Direct Dependency (ALWAYS TRY FIRST)

This is the preferred and recommended approach:

1. **Identify the parent dependency:**
   - Find which direct dependency includes the vulnerable transitive dependency
   - Use ecosystem-specific tools to trace the dependency tree

2. **Check for compatible updates:**
   - Determine if a newer version of the direct dependency includes the patched transitive version
   - Verify compatibility with your project

3. **Update the direct dependency:**
   - Update the direct dependency to the version that resolves the vulnerability
   - This maintains proper dependency management and reduces future conflicts

**Best Practice:** This approach is preferred because it:
- Maintains the natural dependency hierarchy
- Reduces the risk of version conflicts
- Ensures future updates are handled correctly by the package manager
- Avoids manual overrides that may be forgotten

#### Option 2: Use Override Mechanisms (Last resort)

Only use this approach if Option 1 is not viable:

- Use package manager-specific override features when available
- Document the override with a clear explanation
- Include the reason and expected timeline for removal
- Add monitoring to ensure the override is reviewed in future updates

### Example Workflow

```bash
# 1. Check if dependency is transitive
if is_transitive_dependency "$package_name" "$ecosystem"; then
    
    # 2. FIRST: Try to update the direct dependency
    parent_dep=$(find_parent_dependency "$package_name" "$ecosystem")
    
    if can_update_parent "$parent_dep" "$target_version"; then
        echo "Updating direct dependency $parent_dep to resolve transitive vulnerability"
        update_dependency "$parent_dep" "$target_version"
        return 0
    fi
    
    # 3. FALLBACK: If parent update fails, use alternative approaches
    echo "Direct dependency update not viable, using fallback approach"
    # Proceed with Option 2 or 3
fi
```

## Error Handling

### Common Errors

#### 1. Version Not Found

**Error:**
```
No matching version found for {package}@{version}
```

**Solutions:**
- Verify version exists in registry
- Try latest stable version
- Check for typos in version number

#### 2. Dependency Conflicts

**Error:**
```
Dependency conflict detected
```

**Solutions:**
- Update conflicting dependencies together
- Check compatibility matrix
- Review peer dependencies

#### 3. Lock File Out of Sync

**Error:**
```
Lock file out of sync
```

**Solutions:**
- Delete and regenerate lock file
- Run package manager's repair command
- Ensure package manager version is current

#### 4. Network Errors

**Error:**
```
Failed to fetch package
```

**Solutions:**
- Check internet connection
- Verify registry is accessible
- Try alternative registry/mirror

## Best Practices

1. **Always Update Lock Files**
   - Never manually edit lock files
   - Always commit lock files with dependency files

2. **Verify Version Constraints**
   - Preserve existing constraint operators
   - Don't unnecessarily restrict versions

3. **Test After Update**
   - Run build and tests before committing
   - Verify application still works

4. **Commit Atomically**
   - One dependency update per commit
   - Clear commit messages

5. **Document Breaking Changes**
   - Note if update includes breaking changes
   - Reference migration guides

6. **Handle Transitive Carefully**
   - Prefer updating direct dependencies
   - Document why transitive is added directly

## Commit Message Format

### Standard Format

```
Update {package} to {version} to fix security vulnerabilities

- Fixes Dependabot alert #{number}
- Addresses {CVE-ID}: {summary}
- Updated from {old-version} to {new-version}
```

### Example

```
Update library-name to 2.1.5 to fix security vulnerabilities

- Fixes Dependabot alerts #1, #2, #3
- Addresses CVE-2021-12345: Security vulnerability description
- Addresses CVE-2020-67890: Another vulnerability
- Updated from 2.1.0 to 2.1.5
```

## Summary

This guide covers:
- ✅ Ecosystem-agnostic update strategy
- ✅ Auto-detection of package managers
- ✅ Version resolution approaches
- ✅ Transitive dependency handling
- ✅ Verification steps
- ✅ Error handling
- ✅ Best practices

**Key Principle:**
The skill automatically detects the ecosystem and uses the appropriate update commands, making it work seamlessly with any Dependabot-supported package manager.

**Next Steps:**
- Run build and tests (see build-test-verification.md)
- Create PR if successful (see pr-creation.md)
- Create draft PR if tests fail (see migration-guide-discovery.md)

---

*Part of Dependabot Auto-Fix Skill v2.0.0*