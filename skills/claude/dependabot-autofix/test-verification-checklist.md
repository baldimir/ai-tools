# Test Verification Checklist

This checklist ensures thorough verification of dependency updates before creating pull requests.

## Pre-Test Verification

- [ ] Dependency file updated correctly
- [ ] Lock file updated (if applicable)
- [ ] No syntax errors in dependency files
- [ ] Package manager command succeeded
- [ ] No dependency conflicts reported

## Test Command Detection

- [ ] Test command detected automatically OR
- [ ] User provided test command
- [ ] Test command verified to work

## Build Verification (if applicable)

- [ ] Build command detected
- [ ] Build completes successfully
- [ ] No compilation errors
- [ ] Build artifacts created (if expected)
- [ ] No breaking change warnings

## Test Execution

- [ ] Tests run with appropriate timeout (default: 10 minutes)
- [ ] Test output captured completely
- [ ] Exit code checked (0 = success)
- [ ] Test results parsed correctly

## Test Results Analysis

### If Tests Pass ✅

- [ ] All tests passed
- [ ] No new warnings introduced
- [ ] Test coverage maintained (if tracked)
- [ ] Performance acceptable (no significant slowdown)
- [ ] Ready to create PR

### If Tests Fail ❌

- [ ] Test output saved for analysis
- [ ] Failing tests identified
- [ ] Error messages extracted
- [ ] Failure category determined:
  - [ ] Import/Module errors
  - [ ] API signature changes
  - [ ] Behavior changes
  - [ ] Configuration errors
  - [ ] Other
- [ ] Ready to create draft PR with diagnostics

## Post-Test Actions

### For Successful Tests

- [ ] Test output logged
- [ ] Success metrics recorded
- [ ] Proceed to PR creation

### For Failed Tests (Phase 1)

- [ ] Full test output saved
- [ ] Failure analysis completed
- [ ] Diagnostic information prepared
- [ ] Proceed to draft PR creation

## Ecosystem-Specific Checks

### Node.js (npm/yarn/pnpm)

- [ ] `npm test` or equivalent runs successfully
- [ ] No peer dependency warnings
- [ ] Package-lock.json/yarn.lock updated
- [ ] Node version compatibility verified

### Python (pip/pipenv/poetry)

- [ ] `pytest` or equivalent runs successfully
- [ ] No import errors
- [ ] Requirements/lock file updated
- [ ] Python version compatibility verified

### Java (Maven/Gradle)

- [ ] `mvn test` or `gradle test` runs successfully
- [ ] No compilation errors
- [ ] Dependencies resolved correctly
- [ ] Java version compatibility verified

### Ruby (Bundler)

- [ ] `bundle exec rspec` or equivalent runs successfully
- [ ] Gemfile.lock updated
- [ ] No gem conflicts
- [ ] Ruby version compatibility verified

### Go

- [ ] `go test ./...` runs successfully
- [ ] go.sum updated
- [ ] No module errors
- [ ] Go version compatibility verified

### .NET (NuGet)

- [ ] `dotnet test` runs successfully
- [ ] No compilation errors
- [ ] Package references updated
- [ ] Framework compatibility verified

## Common Test Failure Patterns

### Import/Module Errors

**Indicators:**
- `ImportError`, `ModuleNotFoundError`
- `Cannot find module`
- `No module named`

**Actions:**
- Check if imports renamed
- Verify package installed
- Check for moved modules

### API Signature Changes

**Indicators:**
- `TypeError: takes X arguments but Y were given`
- `AttributeError: no attribute`
- `undefined is not a function`

**Actions:**
- Review function signatures
- Check for parameter changes
- Look for deprecated methods

### Behavior Changes

**Indicators:**
- `AssertionError: expected X but got Y`
- Test expectations not met
- Different return values

**Actions:**
- Review test expectations
- Check for behavior changes in release notes
- Verify side effects

### Configuration Errors

**Indicators:**
- `ConfigError`, `ValidationError`
- Invalid configuration format
- Missing required config

**Actions:**
- Check configuration format
- Review config changes in migration guide
- Verify all required fields present

## Timeout Handling

- [ ] Timeout set appropriately (default: 600s)
- [ ] If timeout occurs:
  - [ ] Logged as timeout failure
  - [ ] Considered as test failure
  - [ ] Included in draft PR diagnostics

## Performance Checks

- [ ] Tests complete in reasonable time
- [ ] No significant performance degradation
- [ ] If tests are slower:
  - [ ] Document in PR
  - [ ] Investigate if critical

## Documentation

- [ ] Test results documented
- [ ] Failure analysis documented (if applicable)
- [ ] Diagnostic information prepared for PR
- [ ] Next steps identified

## Final Verification

Before creating PR:

- [ ] All checks completed
- [ ] Test status determined (pass/fail)
- [ ] Appropriate PR type selected (regular/draft)
- [ ] PR description prepared with test results

## Phase 1 Limitations

In Phase 1 (MVP), this checklist focuses on:

- ✅ Running tests
- ✅ Capturing output
- ✅ Analyzing failures
- ✅ Creating draft PRs for failures

Phase 1 does NOT include:

- ❌ Automatic code fixes
- ❌ Retry with modifications
- ❌ Migration guide application

These features are planned for Phase 2.

---

*Part of Dependabot Auto-Fix Skill v1.0.0*