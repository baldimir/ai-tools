# Pull Request Template

This template provides the structure for PR descriptions created by the Dependabot Auto-Fix skill.

## Success PR Template

Use this template when tests pass and the fix is complete.

```markdown
## Summary
This PR fixes {N} Dependabot security alert(s) for `{package-name}`.

## Alerts Fixed

{For each alert:}
- **Alert {number}**: {CVE-ID} - {severity} - {summary}
  - Vulnerable version: {old-version}
  - Fixed version: {new-version}
  - CVSS Score: {score}

## Changes Made
- Updated `{package-name}` from `{old-version}` to `{new-version}`

## Verification
- ✅ Build successful
- ✅ All unit tests passing ({N} tests)

## Additional Notes
{Any relevant notes, warnings, or follow-up items}

---
*This PR was automatically created by the [Dependabot Auto-Fix skill](https://github.com/baldimir/ai-tools).*
```

## Draft PR Template

Use this template when tests fail and manual intervention is required.

```markdown
## ⚠️ Draft PR - Manual Review Required

This PR was automatically created but requires manual intervention to complete.

## Alerts to Fix

{For each alert:}
- **Alert {number}**: {CVE-ID} - {severity} - {summary}
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

**Failure Category**: {Import Error | API Change | Behavior Change | Configuration Error}

**Affected Areas**:
- {List of affected files/modules}

**Suspected Breaking Changes**:
- {List of suspected breaking changes based on error messages}

</details>

## Recommended Next Steps

1. Review the test failures above
2. Check the migration guide: {URL if available, or "Not found"}
3. Manually apply necessary code changes to address:
   - {Specific issue 1}
   - {Specific issue 2}
4. Run tests locally to verify: `{test-command}`
5. Push additional commits to this branch
6. Mark PR as ready for review once tests pass

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
*This draft PR was automatically created by the [Dependabot Auto-Fix skill](https://github.com/baldimir/ai-tools).*
*Tests failed after {N} attempts. Manual intervention required.*
```

## Template Variables

### Common Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `{N}` | Number of alerts | `3` |
| `{package-name}` | Package/library name | `lodash` |
| `{old-version}` | Current version | `4.17.15` |
| `{new-version}` | Target version | `4.17.21` |

### Alert Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `{number}` | Alert number | `1` |
| `{CVE-ID}` | CVE identifier | `CVE-2021-23337` |
| `{severity}` | Severity level | `high` |
| `{summary}` | Brief description | `Command Injection` |
| `{score}` | CVSS score | `7.2` |

### Test Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `{test-command}` | Test command used | `npm test` |
| `{test-name}` | Failing test name | `utils.test.js` |
| `{count}` | Error count | `3` |

### Resource Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `{URL}` | Resource URL | `https://...` |
| `{package-homepage-url}` | Package homepage | `https://lodash.com` |

## Formatting Guidelines

### Severity Badges

Use emoji or text to indicate severity:

- Critical: 🔴 or `critical`
- High: 🟠 or `high`
- Medium: 🟡 or `medium`
- Low: 🟢 or `low`

### Status Indicators

- ✅ Success
- ❌ Failure
- ⚠️ Warning
- ℹ️ Information
- 🔄 In Progress

### Code Blocks

Use appropriate syntax highlighting:

```bash
# For shell commands
npm test
```

```javascript
// For JavaScript code
const result = calculate(2, 3);
```

```python
# For Python code
result = calculate(2, 3)
```

### Collapsible Sections

Use `<details>` for long content:

```markdown
<details>
<summary>Click to expand</summary>

Long content here...

</details>
```

## Example: Success PR

```markdown
## Summary
This PR fixes 3 Dependabot security alerts for `lodash`.

## Alerts Fixed

- **Alert 1**: CVE-2021-23337 - high - Command Injection in lodash
  - Vulnerable version: 4.17.15
  - Fixed version: 4.17.21
  - CVSS Score: 7.2

- **Alert 2**: CVE-2020-28500 - high - Regular Expression Denial of Service (ReDoS)
  - Vulnerable version: 4.17.15
  - Fixed version: 4.17.21
  - CVSS Score: 7.5

- **Alert 3**: CVE-2019-10744 - critical - Prototype Pollution
  - Vulnerable version: 4.17.15
  - Fixed version: 4.17.21
  - CVSS Score: 9.1

## Changes Made
- Updated `lodash` from `4.17.15` to `4.17.21`

## Verification
- ✅ Build successful
- ✅ All unit tests passing (127 tests)

## Additional Notes
This is a patch update with no breaking changes. All tests pass without modification.

---
*This PR was automatically created by the [Dependabot Auto-Fix skill](https://github.com/baldimir/ai-tools).*
```

## Example: Draft PR

```markdown
## ⚠️ Draft PR - Manual Review Required

This PR was automatically created but requires manual intervention to complete.

## Alerts to Fix

- **Alert 5**: CVE-2023-12345 - medium - Security vulnerability in spring-core
  - Vulnerable version: 5.3.15
  - Fixed version: 5.3.20
  - CVSS Score: 6.5

## Attempted Changes
- Updated `spring-core` from `5.3.15` to `5.3.20`

## Issues Encountered

### Test Failures
- `ApplicationContextTest.testBeanCreation`: BeanCreationException
- `WebConfigTest.testMvcConfig`: NoSuchMethodError

### Error Summary
- Import/Module Errors: 0
- API Signature Changes: 2
- Behavior Changes: 0
- Other: 0

## Diagnostic Information

<details>
<summary>Test Output</summary>

```
[ERROR] Tests run: 45, Failures: 2, Errors: 0, Skipped: 0

[ERROR] testBeanCreation(com.example.ApplicationContextTest)
  org.springframework.beans.factory.BeanCreationException: 
  Error creating bean with name 'dataSource'
```
</details>

<details>
<summary>Failure Analysis</summary>

**Failure Category**: API Signature Changes

**Affected Areas**:
- ApplicationContext configuration
- WebMvc configuration

**Suspected Breaking Changes**:
- `Environment.getProperty()` method signature changed
- `WebMvcConfigurer.addResourceHandlers()` method signature changed

</details>

## Recommended Next Steps

1. Review the test failures above
2. Check the migration guide: https://github.com/spring-projects/spring-framework/wiki/Upgrading-to-Spring-Framework-5.x
3. Manually apply necessary code changes
4. Run tests locally: `mvn test`
5. Push additional commits to this branch
6. Mark PR as ready for review once tests pass

## Migration Resources

- Migration guide: https://github.com/spring-projects/spring-framework/wiki/Upgrading-to-Spring-Framework-5.x
- Changelog: https://github.com/spring-projects/spring-framework/releases/tag/v5.3.20

## Manual Fix Hints

**For API Changes:**
- Review function signatures in Spring Framework 5.3.20
- Update method calls to match new signatures
- Check Spring documentation for deprecated methods

---
*This draft PR was automatically created by the [Dependabot Auto-Fix skill](https://github.com/baldimir/ai-tools).*
*Tests failed after 3 attempts. Manual intervention required.*
```

## Best Practices

1. **Be Concise but Complete**
   - Include all necessary information
   - Don't overwhelm with unnecessary details

2. **Use Clear Formatting**
   - Use headings for structure
   - Use lists for readability
   - Use code blocks for commands/output

3. **Provide Context**
   - Explain what was changed
   - Explain why it was changed
   - Link to relevant resources

4. **Include Verification**
   - Show that tests passed
   - Show that build succeeded
   - Provide test counts

5. **For Draft PRs, Be Helpful**
   - Provide clear diagnostics
   - Suggest next steps
   - Link to migration guides
   - Include error analysis

6. **Link to Alerts**
   - Reference alert numbers
   - Include CVE IDs
   - Show severity levels

---

*Part of Dependabot Auto-Fix Skill v1.0.0*