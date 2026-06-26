# Build and Test Verification Guide

This guide provides detailed instructions for verifying that dependency updates don't break the project by running builds and tests.

## Overview

After updating a dependency, it's critical to verify that:
1. The project still builds successfully
2. All tests pass
3. No new errors or warnings are introduced
4. The application functions correctly

## ⚠️ CRITICAL: Security Sanitization

**ALL test output, build logs, and diagnostic information MUST be sanitized before including in PRs or any public documentation.**

### Mandatory Sanitization Steps

1. **Sanitize test output** - Remove credentials, absolute paths, internal IPs
2. **Sanitize build logs** - Remove internal server info, credentials
3. **Sanitize error messages** - Redact sensitive values, connection strings
4. **Sanitize stack traces** - Convert absolute paths to relative paths

### Quick Reference

Before including any output in PR:
```bash
# Sanitize test output
sanitize_test_output() {
    local output="$1"
    
    # Convert absolute paths to relative
    output=$(echo "$output" | sed -E 's|/Users/[^/]+|/[USER_HOME]|g')
    output=$(echo "$output" | sed -E 's|/home/[^/]+|/[USER_HOME]|g')
    
    # Redact tokens/credentials
    output=$(echo "$output" | sed -E 's/[A-Za-z0-9_-]{32,}/[REDACTED_TOKEN]/g')
    
    # Redact URLs with credentials
    output=$(echo "$output" | sed -E 's|://[^:]+:[^@]+@|://[REDACTED]@|g')
    
    # Redact private IPs
    output=$(echo "$output" | sed -E 's/192\.168\.[0-9.]+/[PRIVATE_IP]/g')
    output=$(echo "$output" | sed -E 's/10\.[0-9.]+/[PRIVATE_IP]/g')
    
    echo "$output"
}
```

**See [security-sanitization.md](./security-sanitization.md) for complete sanitization guide.**

## Verification Strategy

### Two-Phase Verification

1. **Build Verification** (if applicable)
   - Compile/transpile code
   - Generate build artifacts
   - Check for compilation errors

2. **Test Verification** (required)
   - Run unit tests
   - Run integration tests (if available)
   - Verify all tests pass

## Test Command Detection

### Priority Order

The skill detects test commands in the following order:

1. **package.json scripts** (Node.js)
2. **Makefile targets**
3. **CI configuration files**
4. **Ecosystem defaults**

### Detection Implementation

```bash
#!/bin/bash

detect_test_command() {
    # 1. Check package.json
    if [ -f "package.json" ]; then
        TEST_CMD=$(jq -r '.scripts.test // empty' package.json)
        if [ -n "$TEST_CMD" ]; then
            echo "npm test"
            return
        fi
    fi
    
    # 2. Check Makefile
    if [ -f "Makefile" ]; then
        if grep -q "^test:" Makefile; then
            echo "make test"
            return
        fi
    fi
    
    # 3. Check CI configuration
    if [ -f ".github/workflows/test.yml" ]; then
        TEST_CMD=$(yq -r '.jobs.test.steps[] | select(.name | contains("test")) | .run' .github/workflows/test.yml | head -1)
        if [ -n "$TEST_CMD" ]; then
            echo "$TEST_CMD"
            return
        fi
    fi
    
    # 4. Use ecosystem defaults
    if [ -f "package.json" ]; then
        echo "npm test"
    elif [ -f "pytest.ini" ] || [ -f "setup.py" ]; then
        echo "pytest"
    elif [ -f "pom.xml" ]; then
        echo "mvn test"
    elif [ -f "build.gradle" ]; then
        echo "gradle test"
    elif [ -f "Gemfile" ]; then
        echo "bundle exec rspec"
    elif [ -f "go.mod" ]; then
        echo "go test ./..."
    elif [ -f "composer.json" ]; then
        echo "composer test"
    elif [ -f "*.csproj" ]; then
        echo "dotnet test"
    else
        echo ""
    fi
}
```

## Build Command Detection

### Priority Order

1. **package.json scripts** (Node.js)
2. **Makefile targets**
3. **CI configuration files**
4. **Ecosystem defaults**

### Detection Implementation

```bash
#!/bin/bash

detect_build_command() {
    # 1. Check package.json
    if [ -f "package.json" ]; then
        BUILD_CMD=$(jq -r '.scripts.build // empty' package.json)
        if [ -n "$BUILD_CMD" ]; then
            echo "npm run build"
            return
        fi
    fi
    
    # 2. Check Makefile
    if [ -f "Makefile" ]; then
        if grep -q "^build:" Makefile; then
            echo "make build"
            return
        fi
    fi
    
    # 3. Check CI configuration
    if [ -f ".github/workflows/build.yml" ]; then
        BUILD_CMD=$(yq -r '.jobs.build.steps[] | select(.name | contains("build")) | .run' .github/workflows/build.yml | head -1)
        if [ -n "$BUILD_CMD" ]; then
            echo "$BUILD_CMD"
            return
        fi
    fi
    
    # 4. Use ecosystem defaults
    if [ -f "package.json" ]; then
        # Check if build script exists
        if jq -e '.scripts.build' package.json > /dev/null; then
            echo "npm run build"
        fi
    elif [ -f "pom.xml" ]; then
        echo "mvn clean install -DskipTests"
    elif [ -f "build.gradle" ]; then
        echo "gradle build -x test"
    elif [ -f "go.mod" ]; then
        echo "go build"
    elif [ -f "*.csproj" ]; then
        echo "dotnet build"
    else
        echo ""
    fi
}
```

## Running Tests

### Basic Test Execution

```bash
#!/bin/bash

run_tests() {
    local test_command="$1"
    local timeout_seconds=600  # 10 minutes
    local output_file="test-output.log"
    
    echo "Running tests: $test_command"
    echo "Timeout: ${timeout_seconds}s"
    
    # Run with timeout and capture output
    if timeout $timeout_seconds bash -c "$test_command" > "$output_file" 2>&1; then
        echo "✅ Tests passed"
        return 0
    else
        local exit_code=$?
        if [ $exit_code -eq 124 ]; then
            echo "⏱️ Tests timed out after ${timeout_seconds}s"
        else
            echo "❌ Tests failed with exit code: $exit_code"
        fi
        return $exit_code
    fi
}
```

### Test Output Capture (with Sanitization)

```bash
#!/bin/bash

capture_test_output() {
    local test_command="$1"
    local output_file="test-output.log"
    local sanitized_output_file="test-output-sanitized.log"
    
    # Run tests and capture all output
    $test_command > "$output_file" 2>&1
    local exit_code=$?
    
    # ⚠️ CRITICAL: Sanitize output before using in PR
    sanitize_test_output "$output_file" > "$sanitized_output_file"
    
    # Store sanitized output for PR inclusion
    cat "$sanitized_output_file"
    
    # Keep raw output for local debugging only (never include in PR)
    # Raw file: test-output.log (DO NOT INCLUDE IN PR)
    # Sanitized file: test-output-sanitized.log (SAFE FOR PR)
    
    return $exit_code
}

sanitize_test_output() {
    local input_file="$1"
    local content=$(cat "$input_file")
    
    # Apply all sanitization patterns
    content=$(echo "$content" | sed -E 's|/Users/[^/]+|/[USER_HOME]|g')
    content=$(echo "$content" | sed -E 's|/home/[^/]+|/[USER_HOME]|g')
    content=$(echo "$content" | sed -E 's|C:\\Users\\[^\\]+|C:\\[USER_HOME]|g')
    content=$(echo "$content" | sed -E 's/[A-Za-z0-9_-]{32,}/[REDACTED_TOKEN]/g')
    content=$(echo "$content" | sed -E 's|://[^:]+:[^@]+@|://[REDACTED]@|g')
    content=$(echo "$content" | sed -E 's/192\.168\.[0-9.]+/[PRIVATE_IP]/g')
    content=$(echo "$content" | sed -E 's/10\.[0-9.]+/[PRIVATE_IP]/g')
    content=$(echo "$content" | sed -E 's/172\.(1[6-9]|2[0-9]|3[01])\.[0-9.]+/[PRIVATE_IP]/g')
    
    echo "$content"
}
```

## Ecosystem-Specific Test Commands

### Node.js (npm/yarn/pnpm)

**npm:**
```bash
npm test
```

**yarn:**
```bash
yarn test
```

**pnpm:**
```bash
pnpm test
```

**Common Test Frameworks:**
- Jest: `jest`
- Mocha: `mocha`
- Jasmine: `jasmine`
- AVA: `ava`

**Example Output (Success):**
```
PASS  src/utils.test.js
  ✓ should calculate sum correctly (2 ms)
  ✓ should handle empty array (1 ms)

Test Suites: 1 passed, 1 total
Tests:       2 passed, 2 total
Snapshots:   0 total
Time:        1.234 s
```

**Example Output (Failure):**
```
FAIL  src/utils.test.js
  ✕ should calculate sum correctly (5 ms)

  ● should calculate sum correctly

    expect(received).toBe(expected)

    Expected: 10
    Received: 15

Test Suites: 1 failed, 1 total
Tests:       1 failed, 1 total
```

---

### Python (pytest/unittest)

**pytest:**
```bash
pytest
# or with verbose output
pytest -v
# or with coverage
pytest --cov
```

**unittest:**
```bash
python -m pytest
# or
python -m unittest discover
```

**Example Output (Success):**
```
============================= test session starts ==============================
collected 10 items

tests/test_utils.py ......                                               [ 60%]
tests/test_models.py ....                                                [100%]

============================== 10 passed in 0.50s ==============================
```

**Example Output (Failure):**
```
============================= test session starts ==============================
collected 10 items

tests/test_utils.py .F....                                               [ 60%]

=================================== FAILURES ===================================
________________________________ test_calculate ________________________________

    def test_calculate():
>       assert calculate(2, 3) == 5
E       assert 6 == 5

tests/test_utils.py:10: AssertionError
========================= 1 failed, 9 passed in 0.50s ==========================
```

---

### Java (Maven)

**Maven:**
```bash
mvn test
# or with clean
mvn clean test
```

**Example Output (Success):**
```
[INFO] -------------------------------------------------------
[INFO]  T E S T S
[INFO] -------------------------------------------------------
[INFO] Running com.example.UtilsTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] Results:
[INFO] 
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] BUILD SUCCESS
```

**Example Output (Failure):**
```
[INFO] -------------------------------------------------------
[INFO]  T E S T S
[INFO] -------------------------------------------------------
[INFO] Running com.example.UtilsTest
[ERROR] Tests run: 5, Failures: 1, Errors: 0, Skipped: 0
[ERROR] 
[ERROR] Results:
[ERROR] 
[ERROR] Failed tests:
[ERROR]   testCalculate(com.example.UtilsTest): expected:<5> but was:<6>
[ERROR] 
[INFO] BUILD FAILURE
```

---

### Java (Gradle)

**Gradle:**
```bash
gradle test
# or with clean
gradle clean test
```

**Example Output (Success):**
```
> Task :test

UtilsTest > testCalculate() PASSED
UtilsTest > testSum() PASSED

BUILD SUCCESSFUL in 2s
5 actionable tasks: 5 executed
```

**Example Output (Failure):**
```
> Task :test FAILED

UtilsTest > testCalculate() FAILED
    org.junit.ComparisonFailure: expected:<5> but was:<6>

5 tests completed, 1 failed

BUILD FAILED in 2s
```

---

### Ruby (RSpec)

**RSpec:**
```bash
bundle exec rspec
# or with format
bundle exec rspec --format documentation
```

**Example Output (Success):**
```
Utils
  #calculate
    should return correct sum
  #multiply
    should return correct product

Finished in 0.01234 seconds
2 examples, 0 failures
```

**Example Output (Failure):**
```
Utils
  #calculate
    should return correct sum (FAILED - 1)

Failures:

  1) Utils#calculate should return correct sum
     Failure/Error: expect(calculate(2, 3)).to eq(5)
     
       expected: 5
            got: 6

Finished in 0.01234 seconds
2 examples, 1 failure
```

---

### Go

**Go Test:**
```bash
go test ./...
# or with verbose output
go test -v ./...
# or with coverage
go test -cover ./...
```

**Example Output (Success):**
```
ok      github.com/user/project/pkg/utils    0.123s
ok      github.com/user/project/pkg/models   0.456s
```

**Example Output (Failure):**
```
--- FAIL: TestCalculate (0.00s)
    utils_test.go:10: Calculate(2, 3) = 6; want 5
FAIL
FAIL    github.com/user/project/pkg/utils    0.123s
```

---

### .NET (NUnit/xUnit)

**dotnet test:**
```bash
dotnet test
# or with verbose output
dotnet test --verbosity normal
```

**Example Output (Success):**
```
Test run for /path/to/project.dll (.NETCoreApp,Version=v6.0)
Microsoft (R) Test Execution Command Line Tool Version 17.0.0

Starting test execution, please wait...
A total of 1 test files matched the specified pattern.

Passed!  - Failed:     0, Passed:     5, Skipped:     0, Total:     5
```

**Example Output (Failure):**
```
Test run for /path/to/project.dll (.NETCoreApp,Version=v6.0)

Starting test execution, please wait...
A total of 1 test files matched the specified pattern.

  Failed TestCalculate [12 ms]
  Error Message:
   Assert.Equal() Failure
Expected: 5
Actual:   6

Failed!  - Failed:     1, Passed:     4, Skipped:     0, Total:     5
```

---

## Test Failure Analysis

### Parsing Test Output (with Sanitization)

```bash
#!/bin/bash

analyze_test_failures() {
    local output_file="$1"
    local sanitized_analysis_file="failure-analysis-sanitized.txt"
    
    echo "Analyzing test failures..."
    
    # Create temporary analysis file
    local temp_analysis=$(mktemp)
    
    # Extract failed test names
    echo "Failed tests:" >> "$temp_analysis"
    grep -E "(FAIL|FAILED|✕|Error)" "$output_file" | head -20 >> "$temp_analysis"
    
    # Extract error messages
    echo "" >> "$temp_analysis"
    echo "Error messages:" >> "$temp_analysis"
    grep -A 5 -E "(Error:|AssertionError|expected|actual)" "$output_file" | head -50 >> "$temp_analysis"
    
    # Categorize failure types
    if grep -q "ImportError\|ModuleNotFoundError\|Cannot find module" "$output_file"; then
        echo "Category: Import/Module Error" >> "$temp_analysis"
    elif grep -q "TypeError\|AttributeError\|undefined is not a function" "$output_file"; then
        echo "Category: API Signature Change" >> "$temp_analysis"
    elif grep -q "AssertionError\|expected.*but was\|Expected.*Received" "$output_file"; then
        echo "Category: Behavior Change" >> "$temp_analysis"
    elif grep -q "SyntaxError\|ParseError" "$output_file"; then
        echo "Category: Syntax Error" >> "$temp_analysis"
    fi
    
    # ⚠️ CRITICAL: Sanitize analysis before using in PR
    sanitize_test_output "$temp_analysis" > "$sanitized_analysis_file"
    
    # Display sanitized analysis
    cat "$sanitized_analysis_file"
    
    # Clean up temp file
    rm -f "$temp_analysis"
    
    echo ""
    echo "⚠️  Sanitized analysis saved to: $sanitized_analysis_file"
    echo "⚠️  Use ONLY the sanitized file in PR descriptions"
}
```

### Failure Categories

**1. Import/Module Errors**
- Missing imports
- Renamed modules
- Moved functions/classes

**Indicators:**
```
ImportError: cannot import name 'X' from 'Y'
ModuleNotFoundError: No module named 'X'
Cannot find module 'X'
```

**2. API Signature Changes**
- Function signature changed
- Different parameter names/order
- Removed/added parameters

**Indicators:**
```
TypeError: X() takes 2 positional arguments but 3 were given
AttributeError: 'X' object has no attribute 'Y'
undefined is not a function
```

**3. Behavior Changes**
- Different return values
- Changed side effects
- Modified defaults

**Indicators:**
```
AssertionError: expected 5 but got 6
Expected: true, Received: false
```

**4. Configuration Errors**
- Invalid configuration format
- Missing required config
- Deprecated config options

**Indicators:**
```
ConfigError: Unknown option 'X'
ValidationError: 'Y' is required
```

---

## Test Timeout Handling

### Setting Timeouts

```bash
#!/bin/bash

run_tests_with_timeout() {
    local test_command="$1"
    local timeout_seconds="${2:-600}"  # Default 10 minutes
    
    echo "Running tests with ${timeout_seconds}s timeout..."
    
    if timeout $timeout_seconds bash -c "$test_command"; then
        echo "✅ Tests completed successfully"
        return 0
    else
        local exit_code=$?
        if [ $exit_code -eq 124 ]; then
            echo "⏱️ Tests timed out after ${timeout_seconds}s"
            echo "Consider:"
            echo "  - Increasing timeout"
            echo "  - Running subset of tests"
            echo "  - Checking for hanging tests"
        fi
        return $exit_code
    fi
}
```

### Recommended Timeouts

| Project Type | Recommended Timeout |
|--------------|---------------------|
| Small library | 2-5 minutes |
| Medium application | 5-10 minutes |
| Large application | 10-20 minutes |
| Integration tests | 15-30 minutes |

---

## Build Verification

### Running Builds (with Sanitization)

```bash
#!/bin/bash

run_build() {
    local build_command="$1"
    local output_file="build-output.log"
    local sanitized_output_file="build-output-sanitized.log"
    
    echo "Running build: $build_command"
    
    if $build_command > "$output_file" 2>&1; then
        echo "✅ Build successful"
        
        # Sanitize build output for PR inclusion
        sanitize_build_output "$output_file" > "$sanitized_output_file"
        
        return 0
    else
        echo "❌ Build failed"
        
        # ⚠️ CRITICAL: Sanitize build output before displaying/including in PR
        sanitize_build_output "$output_file" > "$sanitized_output_file"
        cat "$sanitized_output_file"
        
        echo ""
        echo "⚠️  Raw build output: $output_file (DO NOT INCLUDE IN PR)"
        echo "⚠️  Sanitized output: $sanitized_output_file (SAFE FOR PR)"
        
        return 1
    fi
}

sanitize_build_output() {
    local input_file="$1"
    local content=$(cat "$input_file")
    
    # Apply all sanitization patterns
    content=$(echo "$content" | sed -E 's|/Users/[^/]+|/[USER_HOME]|g')
    content=$(echo "$content" | sed -E 's|/home/[^/]+|/[USER_HOME]|g')
    content=$(echo "$content" | sed -E 's|C:\\Users\\[^\\]+|C:\\[USER_HOME]|g')
    content=$(echo "$content" | sed -E 's/[A-Za-z0-9_-]{32,}/[REDACTED_TOKEN]/g')
    content=$(echo "$content" | sed -E 's|://[^:]+:[^@]+@|://[REDACTED]@|g')
    content=$(echo "$content" | sed -E 's/192\.168\.[0-9.]+/[PRIVATE_IP]/g')
    content=$(echo "$content" | sed -E 's/10\.[0-9.]+/[PRIVATE_IP]/g')
    content=$(echo "$content" | sed -E 's/172\.(1[6-9]|2[0-9]|3[01])\.[0-9.]+/[PRIVATE_IP]/g')
    
    # Redact internal build server information
    content=$(echo "$content" | sed -E 's/jenkins\.[a-z0-9.-]+/[BUILD_SERVER]/g')
    content=$(echo "$content" | sed -E 's/ci\.[a-z0-9.-]+/[BUILD_SERVER]/g')
    
    echo "$content"
}
```

### Build Success Criteria

1. **Exit Code 0**
   - Build command completes successfully

2. **No Compilation Errors**
   - No syntax errors
   - No type errors
   - No missing dependencies

3. **Artifacts Created** (if applicable)
   - dist/ directory populated
   - .jar/.war files created
   - Executables built

4. **No Breaking Warnings**
   - Deprecation warnings are OK
   - Breaking change warnings are not

---

## Complete Verification Workflow

```bash
#!/bin/bash

verify_dependency_update() {
    local package="$1"
    local version="$2"
    
    echo "Verifying dependency update: $package@$version"
    
    # 1. Detect commands
    BUILD_CMD=$(detect_build_command)
    TEST_CMD=$(detect_test_command)
    
    if [ -z "$TEST_CMD" ]; then
        echo "⚠️ No test command detected"
        echo "Please specify test command manually"
        return 1
    fi
    
    # 2. Run build (if applicable)
    if [ -n "$BUILD_CMD" ]; then
        echo "Step 1: Building..."
        if ! run_build "$BUILD_CMD"; then
            echo "❌ Build failed"
            return 1
        fi
        echo "✅ Build successful"
    fi
    
    # 3. Run tests
    echo "Step 2: Running tests..."
    if ! run_tests "$TEST_CMD"; then
        echo "❌ Tests failed"
        
        # Analyze failures
        analyze_test_failures "test-output.log"
        
        return 1
    fi
    echo "✅ All tests passed"
    
    # 4. Success
    echo ""
    echo "✅ Verification complete"
    echo "   - Build: ${BUILD_CMD:-N/A}"
    echo "   - Tests: $TEST_CMD"
    echo "   - Status: PASSED"
    
    return 0
}
```

---

## Handling Test Failures

### Approach: Trigger Migration Guide Discovery and Code Modification

When tests fail:

1. **Capture and Sanitize Full Output**
   ```bash
   # Capture raw output (for local debugging only)
   cat test-output.log > test-failure-details-raw.txt
   
   # ⚠️ CRITICAL: Sanitize before using in PR
   sanitize_test_output test-output.log > test-failure-details-sanitized.txt
   
   # Use ONLY the sanitized version in PR
   ```

2. **Analyze Failure Type (with Sanitization)**
   ```bash
   # Analyze and sanitize in one step
   analyze_test_failures test-output.log > failure-analysis-sanitized.txt
   
   # The analyze_test_failures function now includes sanitization
   ```

3. **Trigger Migration Guide Discovery**
   ```bash
   # Initiate 4-tier search for migration guides
   discover_migration_guide "$PACKAGE_NAME" "$OLD_VERSION" "$NEW_VERSION" "$ECOSYSTEM"
   ```
   
   **Reference:** See `migration-guide-discovery.md` for detailed implementation.
   
   **Search Strategy:**
   - Tier 1: GitHub repository (CHANGELOG, MIGRATION.md, release notes)
   - Tier 2: Package registry (npm, PyPI, Maven, etc.)
   - Tier 3: Official documentation sites
   - Tier 4: Community resources (Stack Overflow, GitHub issues)

4. **Parse Migration Guides**
   ```bash
   # Extract breaking changes and patterns
   parse_migration_guide "migration-relevant.md"
   extract_migration_patterns "migration-relevant.md"
   categorize_changes "breaking-changes.txt"
   ```

5. **Apply Code Modifications**
   ```bash
   # Attempt automatic fixes with retry logic
   intelligent_code_modification "$PACKAGE_NAME" "$OLD_VERSION" "$NEW_VERSION" "$TEST_COMMAND"
   ```
   
   **Reference:** See `code-modification.md` for detailed implementation.
   
   **Fix Strategy:**
   - Attempt 1: Import/module fixes (safest)
   - Attempt 2: Configuration fixes
   - Attempt 3: API signature fixes (more complex)
   - Max 3 attempts with rollback on failure

6. **Verify Fixes**
   ```bash
   # Run tests after each fix attempt
   verify_fixes_with_tests "$TEST_COMMAND"
   compare_test_results "test-output-before.log" "test-output-after.log"
   ```

7. **Handle Results**
   - **If tests pass:** Proceed to Phase 9 (Create PR)
   - **If tests still fail after 3 attempts:** Proceed to Phase 10 (Create Draft PR)
   - **Document all attempts:** Include in PR description

### Workflow Integration

```bash
#!/bin/bash

handle_test_failure() {
    local package_name="$1"
    local old_version="$2"
    local new_version="$3"
    local ecosystem="$4"
    local test_command="$5"
    
    echo "Tests failed - initiating auto-fix workflow..."
    
    # Step 1: Discover migration guides
    if discover_migration_guide "$package_name" "$old_version" "$new_version" "$ecosystem"; then
        echo "✅ Migration guide discovered"
        
        # Step 2: Parse and extract patterns
        parse_migration_guide "migration-relevant.md"
        
        # Step 3: Apply fixes with retry
        if intelligent_code_modification "$package_name" "$old_version" "$new_version" "$test_command"; then
            echo "✅ Auto-fix successful!"
            return 0
        else
            echo "⚠️ Auto-fix failed after max attempts"
        fi
    else
        echo "⚠️ No migration guide found, using fallback strategies"
        
        # Try fixes based on test failure analysis only
        if apply_fixes_with_retry "$package_name" "$test_command"; then
            echo "✅ Fixed without migration guide"
            return 0
        fi
    fi
    
    # If we get here, fixes failed
    echo "❌ Creating draft PR with diagnostics"
    return 1
}
```

---

## Best Practices

1. **Security First - Always Sanitize**
   - **NEVER include raw test output in PRs**
   - **ALWAYS sanitize before including in PR**
   - Use sanitized files only for PR descriptions
   - Keep raw files local for debugging

2. **Always Run Tests Before PR**
   - Never create PR without test verification
   - Tests are the safety net

3. **Capture Full Output (Both Raw and Sanitized)**
   - Save raw output for local debugging
   - Create sanitized version for PR inclusion
   - Clearly label which is which

4. **Set Reasonable Timeouts**
   - Prevent hanging indefinitely
   - Allow enough time for slow tests

5. **Check Build First (if applicable)**
   - Catch compilation errors early
   - Faster feedback than waiting for tests

6. **Analyze Failures Systematically**
   - Categorize error types
   - Identify patterns
   - Document findings
   - **Sanitize analysis before including in PR**

7. **Preserve Test Output (Sanitized Only)**
   - Include ONLY sanitized output in draft PRs
   - Helps manual reviewers
   - Never include raw output with sensitive info

8. **File Naming Convention**
   - Raw files: `*-raw.log` or `*.log` (DO NOT INCLUDE IN PR)
   - Sanitized files: `*-sanitized.log` or `*-safe.log` (SAFE FOR PR)

---

## Troubleshooting

### Issue: No Test Command Found

**Solutions:**
1. Check for test script in package.json
2. Look for Makefile test target
3. Check CI configuration
4. Ask user to specify test command

### Issue: Tests Hang

**Solutions:**
1. Use timeout command
2. Check for infinite loops in tests
3. Check for network calls without timeout
4. Run subset of tests

### Issue: Flaky Tests

**Solutions:**
1. Re-run tests to confirm
2. Document flakiness in PR
3. Consider skipping flaky tests temporarily
4. Report flaky tests to team

### Issue: Environment-Specific Failures

**Solutions:**
1. Check environment variables
2. Verify dependencies installed
3. Check for OS-specific issues
4. Document environment requirements

---

## Summary

This guide covers:
- ✅ **Security sanitization for all output (MANDATORY)**
- ✅ Test command detection strategies
- ✅ Build command detection strategies
- ✅ Running tests with timeout
- ✅ Capturing and analyzing test output (with sanitization)
- ✅ Ecosystem-specific test commands
- ✅ Test failure categorization
- ✅ Complete verification workflow
- ✅ Migration guide discovery on test failure
- ✅ Automatic code modification with retry logic
- ✅ Iterative fix strategy (up to 3 attempts)

**Critical Security Reminders:**
- ⚠️ **ALWAYS sanitize test output before including in PR**
- ⚠️ **ALWAYS sanitize build logs before including in PR**
- ⚠️ **ALWAYS sanitize error messages and stack traces**
- ⚠️ **Use ONLY sanitized files in PR descriptions**

**Next Steps:**
- **Sanitize all output files**
- If tests pass: Create PR (Phase 9) with sanitized diagnostics
- If tests fail: Trigger auto-fix workflow
  - Discover migration guides
  - Apply code modifications
  - Retry tests (up to 3 attempts)
  - **Sanitize all diagnostic information**
- If auto-fix fails: Create draft PR (Phase 10) with sanitized diagnostics

---

*Part of Dependabot Auto-Fix Skill v2.0.0*