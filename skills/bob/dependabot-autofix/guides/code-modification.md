# Code Modification Guide

This guide provides detailed instructions for intelligently analyzing and modifying code to fix breaking changes introduced by dependency updates.

## Overview

After discovering migration guides and identifying breaking changes, the skill must automatically apply code modifications to fix test failures. This guide implements intelligent pattern matching, iterative fixes, and rollback mechanisms.

## When to Use This Guide

Apply code modifications when:
- Tests fail after dependency update
- Migration guide has been discovered
- Breaking changes have been identified
- Automatic fixes are possible

## Code Modification Strategy

### Three-Phase Approach

1. **Analysis Phase**: Understand what needs to change
2. **Modification Phase**: Apply changes systematically
3. **Verification Phase**: Test and iterate

---

## Phase 1: Analysis

### Analyze Test Failures

```bash
#!/bin/bash

analyze_test_failures_for_fixes() {
    local test_output="$1"
    local package_name="$2"
    
    echo "Analyzing test failures to determine fix strategy..."
    
    # Categorize failures
    local failure_type=$(categorize_failure_type "$test_output")
    
    echo "Failure type: $failure_type"
    
    # Extract specific error details
    case "$failure_type" in
        "import_error")
            extract_import_errors "$test_output" "$package_name"
            ;;
        "api_signature_change")
            extract_api_errors "$test_output" "$package_name"
            ;;
        "behavior_change")
            extract_behavior_errors "$test_output" "$package_name"
            ;;
        "config_error")
            extract_config_errors "$test_output" "$package_name"
            ;;
        *)
            echo "⚠️ Unknown failure type"
            ;;
    esac
}

categorize_failure_type() {
    local test_output="$1"
    
    # Check for import/module errors
    if grep -q "ImportError\|ModuleNotFoundError\|Cannot find module\|cannot import name" "$test_output"; then
        echo "import_error"
        return
    fi
    
    # Check for API signature changes
    if grep -q "TypeError.*arguments\|takes.*positional argument\|AttributeError\|undefined is not a function\|NoSuchMethodError" "$test_output"; then
        echo "api_signature_change"
        return
    fi
    
    # Check for behavior changes
    if grep -q "AssertionError\|expected.*but.*got\|Expected:.*Received:" "$test_output"; then
        echo "behavior_change"
        return
    fi
    
    # Check for configuration errors
    if grep -q "ConfigError\|ValidationError\|Invalid configuration\|Unknown option" "$test_output"; then
        echo "config_error"
        return
    fi
    
    echo "unknown"
}

extract_import_errors() {
    local test_output="$1"
    local package_name="$2"
    
    echo "Extracting import errors..."
    
    # Extract the specific import that failed
    grep -E "ImportError|ModuleNotFoundError|Cannot find module" "$test_output" | \
        grep "$package_name" > "import-errors.txt"
    
    # Extract what was being imported
    grep -oP "cannot import name '\K[^']+|from '\K[^']+(?=')" "import-errors.txt" > "failed-imports.txt"
    
    echo "Failed imports:"
    cat "failed-imports.txt"
}

extract_api_errors() {
    local test_output="$1"
    local package_name="$2"
    
    echo "Extracting API signature errors..."
    
    # Extract function/method names that failed
    grep -E "TypeError|AttributeError|NoSuchMethodError" "$test_output" | \
        grep "$package_name" > "api-errors.txt"
    
    # Extract function names
    grep -oP "'\K[^']+(?='\s+(takes|missing|has no attribute))" "api-errors.txt" > "failed-functions.txt"
    
    echo "Failed functions/methods:"
    cat "failed-functions.txt"
}

extract_behavior_errors() {
    local test_output="$1"
    local package_name="$2"
    
    echo "Extracting behavior change errors..."
    
    # Extract assertion failures
    grep -A 3 "AssertionError" "$test_output" > "behavior-errors.txt"
    
    echo "Behavior changes detected in:"
    grep -B 5 "AssertionError" "$test_output" | grep "def test_\|it(" | head -10
}

extract_config_errors() {
    local test_output="$1"
    local package_name="$2"
    
    echo "Extracting configuration errors..."
    
    # Extract config-related errors
    grep -E "ConfigError|ValidationError|Unknown option" "$test_output" > "config-errors.txt"
    
    echo "Configuration issues:"
    cat "config-errors.txt"
}
```

### Find Code Locations

```bash
#!/bin/bash

find_code_to_modify() {
    local package_name="$1"
    local failure_type="$2"
    
    echo "Finding code locations that need modification..."
    
    case "$failure_type" in
        "import_error")
            find_import_locations "$package_name"
            ;;
        "api_signature_change")
            find_api_usage_locations "$package_name"
            ;;
        "config_error")
            find_config_files "$package_name"
            ;;
        *)
            find_all_package_usage "$package_name"
            ;;
    esac
}

find_import_locations() {
    local package_name="$1"
    
    echo "Finding import statements..."
    
    # Search for imports in different languages
    # JavaScript/TypeScript
    grep -rn "import.*from ['\"]${package_name}" . \
        --include="*.js" --include="*.ts" --include="*.jsx" --include="*.tsx" \
        > "import-locations.txt"
    
    # Python
    grep -rn "from ${package_name}\|import ${package_name}" . \
        --include="*.py" \
        >> "import-locations.txt"
    
    # Java
    grep -rn "import ${package_name}" . \
        --include="*.java" \
        >> "import-locations.txt"
    
    echo "Found $(wc -l < import-locations.txt) import locations"
}

find_api_usage_locations() {
    local package_name="$1"
    
    echo "Finding API usage locations..."
    
    # Read failed functions from analysis
    if [ -f "failed-functions.txt" ]; then
        while read -r function_name; do
            echo "Searching for usage of: $function_name"
            grep -rn "$function_name" . \
                --include="*.js" --include="*.ts" --include="*.py" --include="*.java" \
                >> "api-usage-locations.txt"
        done < "failed-functions.txt"
    fi
    
    echo "Found $(wc -l < api-usage-locations.txt) API usage locations"
}

find_config_files() {
    local package_name="$1"
    
    echo "Finding configuration files..."
    
    # Common config file patterns
    find . -type f \( \
        -name "*.config.js" -o \
        -name "*.config.ts" -o \
        -name ".${package_name}rc" -o \
        -name "${package_name}.config.*" -o \
        -name "*.json" -o \
        -name "*.yaml" -o \
        -name "*.yml" \
    \) > "config-files.txt"
    
    echo "Found $(wc -l < config-files.txt) potential config files"
}

find_all_package_usage() {
    local package_name="$1"
    
    echo "Finding all package usage..."
    
    grep -rn "$package_name" . \
        --include="*.js" --include="*.ts" --include="*.py" --include="*.java" \
        --include="*.jsx" --include="*.tsx" \
        > "all-usage-locations.txt"
    
    echo "Found $(wc -l < all-usage-locations.txt) usage locations"
}
```

---

## Phase 2: Modification

### Pattern-Based Fixes

#### 1. Import/Module Fixes

```bash
#!/bin/bash

fix_import_errors() {
    local package_name="$1"
    
    echo "Applying import fixes..."
    
    # Load migration patterns if available
    if [ -f "migration-patterns.json" ]; then
        apply_import_patterns_from_guide "$package_name"
    else
        apply_common_import_fixes "$package_name"
    fi
}

apply_import_patterns_from_guide() {
    local package_name="$1"
    
    echo "Applying import fixes from migration guide..."
    
    # Extract import-related patterns
    jq -r '.[] | select(.type == "import") | {old: .before, new: .after}' migration-patterns.json | \
    while read -r pattern; do
        local old_import=$(echo "$pattern" | jq -r '.old')
        local new_import=$(echo "$pattern" | jq -r '.new')
        
        echo "Replacing: $old_import → $new_import"
        
        # Apply to all files with imports
        while read -r file; do
            apply_import_fix "$file" "$old_import" "$new_import"
        done < import-locations.txt
    done
}

apply_import_fix() {
    local file="$1"
    local old_pattern="$2"
    local new_pattern="$3"
    
    # Extract just the filename from grep output (format: file:line:content)
    local filename=$(echo "$file" | cut -d: -f1)
    
    if [ ! -f "$filename" ]; then
        return
    fi
    
    echo "Modifying: $filename"
    
    # Use sed for simple replacements
    sed -i.bak "s|${old_pattern}|${new_pattern}|g" "$filename"
    
    # Verify the change didn't break syntax
    if ! verify_syntax "$filename"; then
        echo "⚠️ Syntax error after modification, reverting..."
        mv "${filename}.bak" "$filename"
        return 1
    fi
    
    rm -f "${filename}.bak"
    echo "✅ Successfully modified $filename"
}

apply_common_import_fixes() {
    local package_name="$1"
    
    echo "Applying common import fix patterns..."
    
    # Common pattern: named import moved to default import
    # Before: import { something } from 'package'
    # After: import something from 'package'
    
    # Common pattern: submodule reorganization
    # Before: import { X } from 'package/old/path'
    # After: import { X } from 'package/new/path'
    
    # This requires analyzing the actual errors
    if [ -f "failed-imports.txt" ]; then
        while read -r failed_import; do
            echo "Attempting to fix import: $failed_import"
            attempt_import_fix "$package_name" "$failed_import"
        done < "failed-imports.txt"
    fi
}

attempt_import_fix() {
    local package_name="$1"
    local failed_import="$2"
    
    # Try common fixes
    # 1. Check if it moved to a submodule
    # 2. Check if it was renamed
    # 3. Check if it's now a default export
    
    echo "Searching for alternative import path for: $failed_import"
    
    # This is a placeholder - real implementation would be more sophisticated
    # Could use package's index.d.ts or __init__.py to find new location
}
```

#### 2. API Signature Fixes

```bash
#!/bin/bash

fix_api_signature_errors() {
    local package_name="$1"
    
    echo "Applying API signature fixes..."
    
    if [ -f "migration-patterns.json" ]; then
        apply_api_patterns_from_guide "$package_name"
    else
        apply_common_api_fixes "$package_name"
    fi
}

apply_api_patterns_from_guide() {
    local package_name="$1"
    
    echo "Applying API fixes from migration guide..."
    
    # Extract API-related patterns
    jq -r '.[] | select(.type == "api") | {old: .before, new: .after}' migration-patterns.json | \
    while read -r pattern; do
        local old_api=$(echo "$pattern" | jq -r '.old')
        local new_api=$(echo "$pattern" | jq -r '.new')
        
        echo "Replacing API call: $old_api → $new_api"
        
        # Find files using this API
        grep -l "$old_api" api-usage-locations.txt | while read -r file; do
            apply_api_fix "$file" "$old_api" "$new_api"
        done
    done
}

apply_api_fix() {
    local file="$1"
    local old_pattern="$2"
    local new_pattern="$3"
    
    local filename=$(echo "$file" | cut -d: -f1)
    
    if [ ! -f "$filename" ]; then
        return
    fi
    
    echo "Modifying API calls in: $filename"
    
    # For complex API changes, use apply_diff instead of sed
    # This is a simplified example
    sed -i.bak "s|${old_pattern}|${new_pattern}|g" "$filename"
    
    if ! verify_syntax "$filename"; then
        echo "⚠️ Syntax error after modification, reverting..."
        mv "${filename}.bak" "$filename"
        return 1
    fi
    
    rm -f "${filename}.bak"
    echo "✅ Successfully modified $filename"
}

apply_common_api_fixes() {
    local package_name="$1"
    
    echo "Applying common API fix patterns..."
    
    # Common patterns:
    # 1. Function renamed
    # 2. Parameters reordered
    # 3. Parameters changed from positional to named
    # 4. New required parameter added
    
    if [ -f "failed-functions.txt" ]; then
        while read -r function_name; do
            echo "Attempting to fix API usage: $function_name"
            attempt_api_fix "$package_name" "$function_name"
        done < "failed-functions.txt"
    fi
}

attempt_api_fix() {
    local package_name="$1"
    local function_name="$2"
    
    # Analyze the error message to understand what changed
    local error_msg=$(grep "$function_name" api-errors.txt | head -1)
    
    if echo "$error_msg" | grep -q "takes.*positional argument"; then
        echo "Detected: Parameter count mismatch"
        # Would need to analyze the new signature and update calls
    elif echo "$error_msg" | grep -q "has no attribute"; then
        echo "Detected: Function/method renamed or removed"
        # Would need to find the new name from migration guide
    fi
}
```

#### 3. Configuration Fixes

```bash
#!/bin/bash

fix_config_errors() {
    local package_name="$1"
    
    echo "Applying configuration fixes..."
    
    if [ -f "migration-patterns.json" ]; then
        apply_config_patterns_from_guide "$package_name"
    else
        apply_common_config_fixes "$package_name"
    fi
}

apply_config_patterns_from_guide() {
    local package_name="$1"
    
    echo "Applying config fixes from migration guide..."
    
    # Extract config-related patterns
    jq -r '.[] | select(.type == "config") | {old: .before, new: .after}' migration-patterns.json | \
    while read -r pattern; do
        local old_config=$(echo "$pattern" | jq -r '.old')
        local new_config=$(echo "$pattern" | jq -r '.new')
        
        echo "Updating config: $old_config → $new_config"
        
        # Apply to config files
        while read -r config_file; do
            apply_config_fix "$config_file" "$old_config" "$new_config"
        done < config-files.txt
    done
}

apply_config_fix() {
    local file="$1"
    local old_pattern="$2"
    local new_pattern="$3"
    
    if [ ! -f "$file" ]; then
        return
    fi
    
    echo "Modifying config: $file"
    
    # Detect config file type
    local file_ext="${file##*.}"
    
    case "$file_ext" in
        json)
            apply_json_config_fix "$file" "$old_pattern" "$new_pattern"
            ;;
        yaml|yml)
            apply_yaml_config_fix "$file" "$old_pattern" "$new_pattern"
            ;;
        *)
            # Generic text replacement
            sed -i.bak "s|${old_pattern}|${new_pattern}|g" "$file"
            ;;
    esac
    
    if ! verify_config_syntax "$file"; then
        echo "⚠️ Invalid config after modification, reverting..."
        mv "${file}.bak" "$file"
        return 1
    fi
    
    rm -f "${file}.bak"
    echo "✅ Successfully modified $file"
}

apply_json_config_fix() {
    local file="$1"
    local old_key="$2"
    local new_key="$3"
    
    # Use jq to safely modify JSON
    jq "walk(if type == \"object\" and has(\"$old_key\") then . + {\"$new_key\": .[\"$old_key\"]} | del(.[\"$old_key\"]) else . end)" "$file" > "${file}.tmp"
    
    if [ $? -eq 0 ]; then
        mv "${file}.tmp" "$file"
    else
        rm -f "${file}.tmp"
        return 1
    fi
}

apply_yaml_config_fix() {
    local file="$1"
    local old_key="$2"
    local new_key="$3"
    
    # Use sed for YAML (more complex changes would need yq)
    sed -i.bak "s|^${old_key}:|${new_key}:|g" "$file"
}
```

#### 4. Behavior Change Fixes

```bash
#!/bin/bash

fix_behavior_changes() {
    local package_name="$1"
    
    echo "Analyzing behavior changes..."
    
    # Behavior changes are harder to fix automatically
    # Usually require understanding the new behavior and updating tests/code accordingly
    
    if [ -f "behavior-errors.txt" ]; then
        echo "⚠️ Behavior changes detected"
        echo "These typically require manual review:"
        cat "behavior-errors.txt"
        
        # Document for draft PR
        cat > "behavior-changes-summary.md" <<EOF
# Behavior Changes Detected

The following tests are failing due to behavior changes in $package_name:

$(cat behavior-errors.txt)

## Recommended Actions

1. Review the migration guide for behavior changes
2. Update test expectations if the new behavior is correct
3. Update code logic if the old behavior was relied upon
4. Consider if this is a breaking change that requires code updates

EOF
        
        return 1
    fi
    
    return 0
}
```

### Advanced Fix Strategies

#### Using apply_diff for Surgical Changes

```bash
#!/bin/bash

apply_surgical_fix() {
    local file="$1"
    local search_pattern="$2"
    local replace_pattern="$3"
    
    echo "Applying surgical fix to $file..."
    
    # Read the file to find the exact context
    local line_num=$(grep -n "$search_pattern" "$file" | cut -d: -f1 | head -1)
    
    if [ -z "$line_num" ]; then
        echo "⚠️ Pattern not found in file"
        return 1
    fi
    
    # Extract context around the line
    local start_line=$((line_num - 2))
    local end_line=$((line_num + 2))
    
    [ $start_line -lt 1 ] && start_line=1
    
    local context=$(sed -n "${start_line},${end_line}p" "$file")
    
    # Create diff block
    cat > "fix.diff" <<EOF
<<<<<<< SEARCH
:start_line:${start_line}
-------
$context
=======
$(echo "$context" | sed "s|${search_pattern}|${replace_pattern}|")
>>>>>>> REPLACE
EOF
    
    # Apply using Bob's apply_diff (this is conceptual - actual usage would be through Bob)
    echo "Diff prepared for application"
    cat "fix.diff"
}
```

---

## Phase 3: Verification

### Syntax Verification

```bash
#!/bin/bash

verify_syntax() {
    local file="$1"
    local file_ext="${file##*.}"
    
    case "$file_ext" in
        js|jsx)
            node -c "$file" 2>/dev/null
            ;;
        ts|tsx)
            tsc --noEmit "$file" 2>/dev/null
            ;;
        py)
            python -m py_compile "$file" 2>/dev/null
            ;;
        java)
            javac -Xlint:none "$file" 2>/dev/null
            ;;
        *)
            # No syntax check available
            return 0
            ;;
    esac
}

verify_config_syntax() {
    local file="$1"
    local file_ext="${file##*.}"
    
    case "$file_ext" in
        json)
            jq empty "$file" 2>/dev/null
            ;;
        yaml|yml)
            python -c "import yaml; yaml.safe_load(open('$file'))" 2>/dev/null
            ;;
        *)
            return 0
            ;;
    esac
}
```

### Test Verification

```bash
#!/bin/bash

verify_fixes_with_tests() {
    local test_command="$1"
    
    echo "Running tests to verify fixes..."
    
    if timeout 600 bash -c "$test_command" > "test-output-after-fix.log" 2>&1; then
        echo "✅ Tests passed after fixes!"
        return 0
    else
        echo "❌ Tests still failing after fixes"
        
        # Compare before and after
        compare_test_results "test-output.log" "test-output-after-fix.log"
        
        return 1
    fi
}

compare_test_results() {
    local before="$1"
    local after="$2"
    
    echo "Comparing test results..."
    
    local before_failures=$(grep -c "FAIL\|FAILED\|✕" "$before" || echo 0)
    local after_failures=$(grep -c "FAIL\|FAILED\|✕" "$after" || echo 0)
    
    echo "Failures before: $before_failures"
    echo "Failures after: $after_failures"
    
    if [ "$after_failures" -lt "$before_failures" ]; then
        echo "✅ Progress made: $((before_failures - after_failures)) fewer failures"
        return 0
    elif [ "$after_failures" -eq "$before_failures" ]; then
        echo "⚠️ No change in failure count"
        return 1
    else
        echo "⚠️ More failures after fixes - something went wrong"
        return 2
    fi
}
```

---

## Iterative Fix Strategy

### Retry Loop with Max Attempts

```bash
#!/bin/bash

apply_fixes_with_retry() {
    local package_name="$1"
    local test_command="$2"
    local max_attempts=3
    local attempt=1
    
    echo "=========================================="
    echo "Iterative Fix Strategy"
    echo "Package: $package_name"
    echo "Max attempts: $max_attempts"
    echo "=========================================="
    
    while [ $attempt -le $max_attempts ]; do
        echo ""
        echo "Attempt $attempt of $max_attempts"
        echo "----------------------------------------"
        
        # Analyze current test failures
        analyze_test_failures_for_fixes "test-output.log" "$package_name"
        
        # Determine fix strategy based on failure type
        local failure_type=$(cat failure-category.txt 2>/dev/null || echo "unknown")
        
        echo "Applying fixes for: $failure_type"
        
        # Apply fixes based on type
        case "$failure_type" in
            "import_errors")
                fix_import_errors "$package_name"
                ;;
            "api_changes")
                fix_api_signature_errors "$package_name"
                ;;
            "config_error")
                fix_config_errors "$package_name"
                ;;
            "behavior_changes")
                if ! fix_behavior_changes "$package_name"; then
                    echo "⚠️ Behavior changes require manual intervention"
                    return 1
                fi
                ;;
            *)
                echo "⚠️ Unknown failure type, trying generic fixes"
                apply_generic_fixes "$package_name"
                ;;
        esac
        
        # Commit the changes
        git add -A
        git commit -m "Attempt $attempt: Fix $failure_type for $package_name" || true
        
        # Run tests again
        echo ""
        echo "Running tests after fixes..."
        
        if verify_fixes_with_tests "$test_command"; then
            echo ""
            echo "✅ SUCCESS! Tests passing after $attempt attempt(s)"
            return 0
        fi
        
        # Check if we made progress
        if compare_test_results "test-output.log" "test-output-after-fix.log"; then
            echo "Progress made, continuing to next attempt..."
            mv "test-output-after-fix.log" "test-output.log"
        else
            echo "No progress made, trying different strategy..."
        fi
        
        attempt=$((attempt + 1))
    done
    
    echo ""
    echo "❌ Failed to fix issues after $max_attempts attempts"
    echo "Creating draft PR with diagnostic information..."
    
    return 1
}

apply_generic_fixes() {
    local package_name="$1"
    
    echo "Applying generic fix strategies..."
    
    # Try all fix types
    fix_import_errors "$package_name" || true
    fix_api_signature_errors "$package_name" || true
    fix_config_errors "$package_name" || true
}
```

### Progressive Fix Strategy

```bash
#!/bin/bash

apply_progressive_fixes() {
    local package_name="$1"
    local test_command="$2"
    
    echo "Using progressive fix strategy..."
    
    # Start with safest fixes first
    
    # Level 1: Import fixes (safest)
    echo "Level 1: Fixing imports..."
    fix_import_errors "$package_name"
    git add -A && git commit -m "Fix imports for $package_name" || true
    
    if verify_fixes_with_tests "$test_command"; then
        echo "✅ Fixed with import changes only"
        return 0
    fi
    
    # Level 2: Configuration fixes
    echo "Level 2: Fixing configuration..."
    fix_config_errors "$package_name"
    git add -A && git commit -m "Fix configuration for $package_name" || true
    
    if verify_fixes_with_tests "$test_command"; then
        echo "✅ Fixed with import + config changes"
        return 0
    fi
    
    # Level 3: API signature fixes (more risky)
    echo "Level 3: Fixing API signatures..."
    fix_api_signature_errors "$package_name"
    git add -A && git commit -m "Fix API signatures for $package_name" || true
    
    if verify_fixes_with_tests "$test_command"; then
        echo "✅ Fixed with all automated changes"
        return 0
    fi
    
    # Level 4: Behavior changes (requires manual review)
    echo "Level 4: Behavior changes detected - manual review required"
    fix_behavior_changes "$package_name"
    
    return 1
}
```

---

## Rollback Mechanism

### Rollback on Failure

```bash
#!/bin/bash

rollback_changes() {
    local reason="$1"
    
    echo "=========================================="
    echo "Rolling back changes"
    echo "Reason: $reason"
    echo "=========================================="
    
    # Check if we have commits to rollback
    local commits_since_start=$(git log --oneline --since="10 minutes ago" | wc -l)
    
    if [ "$commits_since_start" -gt 0 ]; then
        echo "Rolling back $commits_since_start commit(s)..."
        git reset --hard HEAD~"$commits_since_start"
        echo "✅ Rollback complete"
    else
        echo "No commits to rollback"
    fi
    
    # Clean up any uncommitted changes
    git checkout -- .
    git clean -fd
    
    echo "Repository restored to pre-fix state"
}

safe_apply_with_rollback() {
    local package_name="$1"
    local test_command="$2"
    
    # Save current state
    local start_commit=$(git rev-parse HEAD)
    
    echo "Starting fixes from commit: $start_commit"
    
    # Try to apply fixes
    if apply_fixes_with_retry "$package_name" "$test_command"; then
        echo "✅ Fixes successful, keeping changes"
        return 0
    else
        echo "❌ Fixes failed, rolling back..."
        git reset --hard "$start_commit"
        echo "✅ Rolled back to: $start_commit"
        return 1
    fi
}
```

---

## Language-Agnostic Fix Strategies

The skill uses **general patterns** that work across programming languages:

### Common Fix Patterns

1. **Import/Module Changes**
   - Detect import errors from test output
   - Search for old import patterns
   - Replace with new import patterns from migration guide

2. **API Signature Changes**
   - Identify function/method calls that fail
   - Update parameter lists
   - Adjust return value handling

3. **Configuration Changes**
   - Locate configuration files
   - Update deprecated options
   - Add new required settings

4. **Behavior Changes**
   - Analyze test failures
   - Adjust code expectations
   - Update assertions

### General Fix Approach

```bash
#!/bin/bash

apply_fixes() {
    local package_name="$1"
    local failure_type="$2"
    
    echo "Applying general fixes for $failure_type..."
    
    # 1. Analyze test failures
    analyze_test_output "$package_name"
    
    # 2. Find affected code locations
    find_code_locations "$package_name"
    
    # 3. Apply appropriate fixes
    case "$failure_type" in
        import_errors)
            fix_import_patterns "$package_name"
            ;;
        api_changes)
            fix_api_calls "$package_name"
            ;;
        config_changes)
            fix_configuration "$package_name"
            ;;
        *)
            echo "Applying general fixes..."
            ;;
    esac
    
    # 4. Verify syntax
    verify_code_syntax
}
```

### Language Detection

The skill automatically detects the programming language and adapts its approach:

```bash
detect_language() {
    # Auto-detect from project files
    if [ -f "package.json" ]; then
        echo "javascript"
    elif [ -f "requirements.txt" ] || [ -f "setup.py" ]; then
        echo "python"
    elif [ -f "pom.xml" ] || [ -f "build.gradle" ]; then
        echo "java"
    elif [ -f "Gemfile" ]; then
        echo "ruby"
    elif [ -f "go.mod" ]; then
        echo "go"
    elif [ -f "Cargo.toml" ]; then
        echo "rust"
    else
        echo "unknown"
    fi
}
```

---

## Documentation and Reporting

### Document Applied Fixes

```bash
#!/bin/bash

document_applied_fixes() {
    local package_name="$1"
    
    cat > "fixes-applied.md" <<EOF
# Code Modifications Applied

## Package: $package_name

## Fixes Applied

$(git log --oneline --since="10 minutes ago" | grep -i "fix\|attempt")

## Files Modified

$(git diff --name-only HEAD~3..HEAD 2>/dev/null || echo "No changes")

## Fix Categories

$([ -f "import-locations.txt" ] && echo "- Import fixes: $(wc -l < import-locations.txt) locations")
$([ -f "api-usage-locations.txt" ] && echo "- API fixes: $(wc -l < api-usage-locations.txt) locations")
$([ -f "config-files.txt" ] && echo "- Config fixes: $(wc -l < config-files.txt) files")

## Test Results

Before fixes:
$(grep -c "FAIL\|FAILED" test-output.log 2>/dev/null || echo "0") failures

After fixes:
$(grep -c "FAIL\|FAILED" test-output-after-fix.log 2>/dev/null || echo "0") failures

## Migration Guide Used

$([ -f "migration-relevant.md" ] && echo "✅ Migration guide found and used" || echo "⚠️ No migration guide found")

EOF

    echo "✅ Fix documentation created: fixes-applied.md"
}
```

---

## Complete Workflow

```bash
#!/bin/bash

intelligent_code_modification() {
    local package_name="$1"
    local old_version="$2"
    local new_version="$3"
    local test_command="$4"
    
    echo "=========================================="
    echo "Intelligent Code Modification"
    echo "Package: $package_name"
    echo "Version: $old_version → $new_version"
    echo "=========================================="
    
    # Step 1: Analyze test failures
    echo ""
    echo "Step 1: Analyzing test failures..."
    analyze_test_failures_for_fixes "test-output.log" "$package_name"
    
    # Step 2: Find code locations
    echo ""
    echo "Step 2: Finding code to modify..."
    local failure_type=$(cat failure-category.txt 2>/dev/null || echo "unknown")
    find_code_to_modify "$package_name" "$failure_type"
    
    # Step 3: Apply fixes with retry
    echo ""
    echo "Step 3: Applying fixes..."
    if safe_apply_with_rollback "$package_name" "$test_command"; then
        echo ""
        echo "✅ Code modification successful!"
        
        # Document what was done
        document_applied_fixes "$package_name"
        
        return 0
    else
        echo ""
        echo "❌ Code modification failed"
        echo "Creating draft PR with diagnostic information..."
        
        # Document attempted fixes
        document_applied_fixes "$package_name"
        
        return 1
    fi
}
```

---

## Best Practices

1. **Start with Safest Fixes**
   - Import fixes first
   - Configuration fixes second
   - API changes last

2. **Verify After Each Change**
   - Run syntax checks
   - Run tests
   - Compare results

3. **Use Version Control**
   - Commit after each fix attempt
   - Easy to rollback if needed
   - Track what was tried

4. **Document Everything**
   - What was changed
   - Why it was changed
   - Results of changes

5. **Know When to Stop**
   - Max 3 attempts
   - If no progress, stop
   - Create draft PR for manual review

6. **Preserve Context**
   - Save all analysis
   - Save test outputs
   - Include in PR description

---

## Error Handling

```bash
#!/bin/bash

handle_modification_errors() {
    local error_type="$1"
    
    case "$error_type" in
        "syntax_error")
            echo "⚠️ Syntax error after modification"
            echo "Rolling back changes..."
            ;;
        "no_progress")
            echo "⚠️ No progress after multiple attempts"
            echo "Manual intervention required"
            ;;
        "worse_results")
            echo "⚠️ More failures after fixes"
            echo "Rolling back to previous state"
            ;;
        "timeout")
            echo "⚠️ Tests timed out"
            echo "May need to increase timeout or fix hanging tests"
            ;;
        *)
            echo "⚠️ Unknown error during modification"
            ;;
    esac
}
```

---

## Summary

This guide covers:
- ✅ Intelligent test failure analysis
- ✅ Pattern-based code modifications
- ✅ Import, API, config, and behavior fixes
- ✅ Iterative fix strategy with retry logic
- ✅ Rollback mechanisms for failed fixes
- ✅ Language-specific strategies
- ✅ Verification and documentation
- ✅ Complete workflow integration

**Next Steps:**
- If fixes successful: Create PR (Phase 9)
- If fixes failed: Create draft PR (Phase 10)
- Document all changes in PR description

---

*Part of Dependabot Auto-Fix Skill v2.0.0*