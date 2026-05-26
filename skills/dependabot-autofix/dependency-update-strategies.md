# Dependency Update Strategies

This guide provides a quick reference for dependency update approaches that work across all package ecosystems.

## General Strategy

The skill uses an **ecosystem-agnostic approach** that automatically:

1. **Detects** the package manager from project files
2. **Identifies** the appropriate dependency and lock files
3. **Updates** using the correct commands for that ecosystem
4. **Verifies** changes are applied correctly

## Update Process

### Step-by-Step

1. **Auto-Detection**
   - Scan for package manager files
   - Identify ecosystem from alert data
   - Locate dependency files

2. **Version Resolution**
   - Use patched version from alert
   - Fallback to latest stable if needed
   - Preserve version constraints

3. **Update Execution**
   - Modify dependency file
   - Run package manager update command
   - Verify lock file changes

4. **Verification**
   - Check dependency file updated
   - Check lock file updated (if applicable)
   - Verify no conflicts introduced

5. **Commit**
   - Stage all changed files
   - Create descriptive commit message

## Handling Different Dependency Types

### Direct Dependencies

**Approach:** Update dependency file directly

**Steps:**
1. Locate dependency declaration
2. Update version number
3. Run package manager update
4. Verify lock file updated
5. Commit changes

### Transitive Dependencies

**Approach:** Update parent dependency or add as direct

**Steps:**
1. Identify which direct dependency includes the vulnerable transitive
2. Check if updating direct dependency fixes the issue
3. If yes: Update direct dependency
4. If no: Add transitive as direct dependency with comment
5. Alternative: Use override mechanisms (if available)

## Version Constraint Preservation

The skill preserves existing version constraints:

- Caret operators (`^`) - Compatible versions
- Tilde operators (`~`) - Approximately equivalent
- Comparison operators (`>=`, `>`, `<`, `<=`) - Range constraints
- Exact versions - No operator

**Example:**
```
Before: package@^1.2.3
After:  package@^1.2.5
(Constraint operator preserved)
```

## Verification Checklist

After updating a dependency:

- [ ] Dependency file updated
- [ ] Lock file updated (if applicable)
- [ ] Version constraint preserved
- [ ] Package manager command succeeded
- [ ] No dependency conflicts
- [ ] Changes committed

## Common Issues

### Issue: Version Not Found

**Solutions:**
- Verify version exists in registry
- Check for typos
- Try latest stable version
- Check package name spelling

### Issue: Dependency Conflicts

**Solutions:**
- Update conflicting dependencies together
- Check compatibility matrix
- Review peer dependencies
- Use force flag (with caution)

### Issue: Lock File Out of Sync

**Solutions:**
- Delete and regenerate lock file
- Run package manager repair command
- Ensure package manager version is current

## Best Practices

1. **One Dependency at a Time**
   - Update one library per PR
   - Easier to track and rollback

2. **Preserve Constraints**
   - Don't unnecessarily restrict versions
   - Keep existing constraint operators

3. **Update Lock Files**
   - Always commit lock files
   - Never manually edit lock files

4. **Test After Update**
   - Run tests before committing
   - Verify application still works

5. **Document Changes**
   - Clear commit messages
   - Reference CVE/alert numbers

## Ecosystem Support

This strategy works with **any ecosystem Dependabot supports**, including:
- Node.js (npm, yarn, pnpm)
- Python (pip, pipenv, poetry)
- Java (Maven, Gradle)
- Ruby (Bundler)
- PHP (Composer)
- Go (Go modules)
- .NET (NuGet)
- Rust (Cargo)
- And more...

The skill automatically adapts to the detected ecosystem without requiring ecosystem-specific configuration.

---

*Part of Dependabot Auto-Fix Skill v2.0.0*