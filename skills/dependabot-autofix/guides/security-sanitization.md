# Security Sanitization Guide

## Overview

This guide provides comprehensive instructions for sanitizing all content before including it in Pull Requests. **Security sanitization is MANDATORY** and must be performed on all PR descriptions, comments, diagnostics, test outputs, and any other content that will be publicly visible.

## Critical Security Principle

**NEVER include sensitive information in PRs.** When in doubt, redact. It's better to provide less information than to leak sensitive data.

## Sensitive Information Categories

### 1. Credentials and Secrets

**What to Detect:**
- API keys and tokens
- Passwords and passphrases
- OAuth tokens and refresh tokens
- SSH keys and certificates
- Database credentials
- Service account credentials
- JWT tokens
- Session tokens

**Patterns to Detect:**
```regex
# Generic API keys/tokens (20+ alphanumeric characters)
[A-Za-z0-9_\-]{20,}

# Common API key formats
(api[_-]?key|apikey|api[_-]?token)[=:\s]+[A-Za-z0-9_\-]+

# AWS keys
AKIA[0-9A-Z]{16}

# GitHub tokens
gh[pousr]_[A-Za-z0-9]{36,}

# Generic secrets
(secret|password|passwd|pwd)[=:\s]+\S+

# Bearer tokens
Bearer\s+[A-Za-z0-9\-._~+/]+=*
```

**How to Redact:**
```
Before: API_KEY=[EXAMPLE_API_KEY]
After:  API_KEY=[REDACTED_API_KEY]

Before: Authorization: Bearer [EXAMPLE_JWT_TOKEN]
After:  Authorization: Bearer [REDACTED_TOKEN]

Before: password: [EXAMPLE_PASSWORD]
After:  password: [REDACTED]
```

### 2. Internal File Paths

**What to Detect:**
- Absolute file paths
- Home directory paths
- Username-specific paths
- System-specific paths

**Patterns to Detect:**
```regex
# Unix/Linux/macOS home directories
/home/[^/\s]+/
/Users/[^/\s]+/

# Windows paths
C:\\Users\\[^\\]+\\
[A-Z]:\\[^\\]+\\

# Generic absolute paths
^/[a-z]+/[a-z]+/
```

**How to Redact:**
```
Before: /Users/[USERNAME]/projects/myapp/src/main.ts
After:  /[USER_HOME]/projects/myapp/src/main.ts
OR:     ./src/main.ts (use relative paths)

Before: C:\Users\[USERNAME]\workspace\app\config.json
After:  ./config.json

Before: /home/[USERNAME]/build/output.log
After:  ./build/output.log
```

### 3. Environment Variables

**What to Detect:**
- Environment variable values (not names)
- Configuration values
- Runtime parameters

**Patterns to Detect:**
```regex
# Environment variable assignments
[A-Z_][A-Z0-9_]*=[^\s]+

# Export statements
export\s+[A-Z_][A-Z0-9_]*=[^\s]+
```

**How to Redact:**
```
Before: DATABASE_URL=postgresql://[EXAMPLE_USER]:[EXAMPLE_PASS]@localhost:5432/db
After:  DATABASE_URL=[REDACTED]

Before: export AWS_SECRET_ACCESS_KEY=[EXAMPLE_AWS_SECRET_KEY]
After:  export AWS_SECRET_ACCESS_KEY=[REDACTED]

Before: NODE_ENV=production API_KEY=[EXAMPLE_KEY] PORT=3000
After:  NODE_ENV=production API_KEY=[REDACTED] PORT=3000
```

**Safe to Include:**
- Environment variable names (e.g., `DATABASE_URL`, `API_KEY`)
- Non-sensitive values (e.g., `NODE_ENV=production`, `PORT=3000`)

### 4. URLs with Authentication

**What to Detect:**
- URLs containing credentials
- URLs with tokens in query parameters
- URLs with authentication in the path

**Patterns to Detect:**
```regex
# URLs with username:password
[a-z]+://[^:]+:[^@]+@

# URLs with tokens in query params
[?&](token|key|secret|auth|api_key)=[^&\s]+

# URLs with tokens in path
/api/[a-z]+/[A-Za-z0-9_\-]{20,}
```

**How to Redact:**
```
Before: https://[EXAMPLE_USER]:[EXAMPLE_PASS]@api.example.com/data
After:  https://[REDACTED]@api.example.com/data

Before: https://api.example.com/data?api_key=[EXAMPLE_API_KEY]
After:  https://api.example.com/data?api_key=[REDACTED]

Before: git clone https://oauth2:[EXAMPLE_TOKEN]@github.com/org/repo.git
After:  git clone https://[REDACTED]@github.com/org/repo.git
```

### 5. Private Network Information

**What to Detect:**
- Private IP addresses
- Internal hostnames
- MAC addresses
- Internal DNS names

**Patterns to Detect:**
```regex
# Private IPv4 ranges
10\.\d{1,3}\.\d{1,3}\.\d{1,3}
172\.(1[6-9]|2[0-9]|3[01])\.\d{1,3}\.\d{1,3}
192\.168\.\d{1,3}\.\d{1,3}

# Localhost variations
127\.\d{1,3}\.\d{1,3}\.\d{1,3}

# MAC addresses
([0-9A-Fa-f]{2}[:-]){5}([0-9A-Fa-f]{2})

# Internal hostnames
[a-z0-9\-]+\.internal
[a-z0-9\-]+\.local
```

**How to Redact:**
```
Before: Connecting to 10.0.0.100:5432
After:  Connecting to [PRIVATE_IP]:5432

Before: Server: database.example.com
After:  Server: [INTERNAL_HOST]

Before: MAC: [EXAMPLE_MAC_ADDRESS]
After:  MAC: [REDACTED_MAC]
```

### 6. Stack Traces and Error Messages

**What to Detect:**
- Full stack traces with absolute paths
- Error messages containing sensitive values
- Debug output with internal information

**How to Sanitize:**
```
Before:
Error: Connection failed
    at Database.connect (/Users/[USERNAME]/projects/app/src/db.ts:45:12)
    at Server.start (/Users/[USERNAME]/projects/app/src/server.ts:23:8)
    at main (/Users/[USERNAME]/projects/app/src/index.ts:10:5)

After:
Error: Connection failed
    at Database.connect (./src/db.ts:45:12)
    at Server.start (./src/server.ts:23:8)
    at main (./src/index.ts:10:5)
```

**For Error Messages:**
```
Before: Error: Failed to connect to postgresql://[EXAMPLE_USER]:[EXAMPLE_PASS]@database.example.com:5432/prod
After:  Error: Failed to connect to database [connection details redacted]

Before: Authentication failed for user 'user@example.com' with token '[EXAMPLE_TOKEN]'
After:  Authentication failed [credentials redacted]
```

### 7. Configuration Values

**What to Detect:**
- Database connection strings
- Service URLs with credentials
- API endpoints with tokens
- Configuration file contents

**How to Redact:**
```
Before:
{
  "database": "postgresql://[EXAMPLE_USER]:[EXAMPLE_PASS]@database.example.com:5432/prod",
  "redis": "redis://:[EXAMPLE_PASS]@cache.example.com:6379",
  "apiKey": "[EXAMPLE_API_KEY]"
}

After:
{
  "database": "[REDACTED_CONNECTION_STRING]",
  "redis": "[REDACTED_CONNECTION_STRING]",
  "apiKey": "[REDACTED]"
}
```

### 8. Git and Version Control Information

**What to Detect:**
- Internal email addresses
- Commit messages with sensitive info
- Branch names with sensitive info

**How to Redact:**
```
Before: Author: user@example.com
After:  Author: [REDACTED_EMAIL]

Before: Commit: "Added API key [EXAMPLE_KEY] for production"
After:  Commit: "Added API key [REDACTED] for production"
```

**Safe to Include:**
- Public email addresses (e.g., GitHub noreply emails)
- Generic commit messages
- Public branch names

## Sanitization Strategies

### Strategy 1: Pattern-Based Replacement

Use regex patterns to detect and replace sensitive information:

```bash
# Example sanitization script
sanitize_content() {
    local content="$1"
    
    # Redact API keys/tokens
    content=$(echo "$content" | sed -E 's/\[EXAMPLE_[A-Z_]+\]/[REDACTED_TOKEN]/g')
    
    # Redact absolute paths
    content=$(echo "$content" | sed -E 's|/Users/\[USERNAME\]|/[USER_HOME]|g')
    content=$(echo "$content" | sed -E 's|/home/\[USERNAME\]|/[USER_HOME]|g')
    
    # Redact URLs with credentials
    content=$(echo "$content" | sed -E 's|://\[EXAMPLE_[A-Z_]+\]:\[EXAMPLE_[A-Z_]+\]@|://[REDACTED]@|g')
    
    # Redact private IPs
    content=$(echo "$content" | sed -E 's/10\.0\.0\.[0-9]{1,3}/[PRIVATE_IP]/g')
    
    echo "$content"
}
```

### Strategy 2: Allowlist Approach

Only include known-safe information:

```
SAFE_TO_INCLUDE:
- Dependency names and versions
- CVE identifiers (CVE-XXXX-XXXXX)
- Public error types (without values)
- Test names (without output)
- Relative file paths
- Public documentation URLs
- Public package registry URLs
```

### Strategy 3: Manual Review

Always manually review sanitized content before creating PR:

1. Read through entire PR description
2. Check for any patterns that look like credentials
3. Verify all paths are relative
4. Ensure no internal hostnames or IPs
5. Confirm no environment variable values
6. Check URLs for embedded credentials

## Sanitization Checklist

Before creating any PR, verify:

- [ ] All absolute paths converted to relative paths
- [ ] No API keys, tokens, or passwords visible
- [ ] No environment variable values (names OK)
- [ ] No URLs with embedded credentials
- [ ] No private IP addresses or internal hostnames
- [ ] Stack traces sanitized (relative paths only)
- [ ] Error messages don't contain sensitive values
- [ ] No database connection strings
- [ ] No internal email addresses
- [ ] No MAC addresses or network identifiers
- [ ] Test output sanitized
- [ ] Build logs sanitized
- [ ] Configuration values redacted

## Examples

### Example 1: Sanitized Test Output

**Before:**
```
Running tests...
✓ should connect to database (postgresql://[EXAMPLE_USER]:[EXAMPLE_PASS]@10.0.0.50:5432/test)
✗ should authenticate user (API key: [EXAMPLE_API_KEY] failed)
✓ should load config from /Users/[USERNAME]/projects/app/config/test.json
```

**After:**
```
Running tests...
✓ should connect to database
✗ should authenticate user (authentication failed)
✓ should load config from ./config/test.json
```

### Example 2: Sanitized Error Message

**Before:**
```
Error: Failed to update dependency
  Caused by: Network error connecting to https://[EXAMPLE_KEY]:[EXAMPLE_TOKEN]@registry.example.com
  Stack trace:
    at updateDependency (/Users/[USERNAME]/workspace/app/src/updater.ts:89:15)
    at main (/Users/[USERNAME]/workspace/app/src/index.ts:12:8)
  Environment: NODE_ENV=production, API_KEY=[EXAMPLE_API_KEY], DATABASE_URL=postgresql://[EXAMPLE_USER]:[EXAMPLE_PASS]@database.example.com:5432/prod
```

**After:**
```
Error: Failed to update dependency
  Caused by: Network error connecting to registry
  Stack trace:
    at updateDependency (./src/updater.ts:89:15)
    at main (./src/index.ts:12:8)
  Environment: NODE_ENV=production, API_KEY=[REDACTED], DATABASE_URL=[REDACTED]
```

### Example 3: Sanitized PR Description

**Before:**
```
## Changes
Updated lodash from 4.17.20 to 4.17.21 to fix CVE-2021-23337

## Test Results
All tests passed on build server ci-server.example.com
Build log: /var/jenkins/jobs/myapp/builds/123/log.txt
Test database: postgresql://[EXAMPLE_USER]:[EXAMPLE_PASS]@10.0.0.100:5432/testdb

## Verification
Deployed to staging: https://[EXAMPLE_USER]:[EXAMPLE_PASS]@staging.example.com
API endpoint: https://api.example.com/v1?token=[EXAMPLE_TOKEN]
```

**After:**
```
## Changes
Updated lodash from 4.17.20 to 4.17.21 to fix CVE-2021-23337

## Test Results
All tests passed on CI server
Build completed successfully
Test database connection verified

## Verification
Deployed to staging environment
API endpoint verified
```

## Safe Information to Include

The following information is generally safe to include in PRs:

### ✅ Safe
- Dependency names (e.g., `lodash`, `react`, `express`)
- Dependency versions (e.g., `4.17.21`, `^18.0.0`)
- CVE identifiers (e.g., `CVE-2021-23337`)
- Public error types (e.g., `TypeError`, `ReferenceError`)
- Test names (e.g., `should validate input`)
- Relative file paths (e.g., `./src/utils.ts`)
- Public documentation URLs (e.g., `https://docs.npmjs.com`)
- Public package registry URLs (e.g., `https://registry.npmjs.org`)
- Generic environment names (e.g., `production`, `staging`)
- Public GitHub repository URLs
- Standard port numbers (e.g., `3000`, `8080`)

### ❌ Never Include
- Actual credentials or tokens
- Absolute file paths with usernames
- Internal hostnames or IP addresses
- Environment variable values
- Database connection strings
- URLs with embedded credentials
- Full stack traces with absolute paths
- Internal email addresses
- Configuration file contents with secrets

## Implementation Notes

### When to Sanitize

Sanitize content at these points:

1. **Before creating PR description** - Sanitize all content
2. **Before adding PR comments** - Sanitize diagnostic information
3. **Before including test output** - Sanitize test results
4. **Before including build logs** - Sanitize build output
5. **Before including error messages** - Sanitize error details

### How to Sanitize

1. **Read the content** - Understand what you're sanitizing
2. **Apply pattern-based replacements** - Use regex for common patterns
3. **Manual review** - Check for anything that looks sensitive
4. **Verify relative paths** - Ensure all paths are relative
5. **Test the output** - Make sure sanitized content is still useful

### Sanitization Tools

Consider using these approaches:

```bash
# Bash function for sanitization
sanitize_for_pr() {
    local input="$1"
    
    # Apply all sanitization patterns
    input=$(echo "$input" | sed -E 's|/Users/\[USERNAME\]|/[USER_HOME]|g')
    input=$(echo "$input" | sed -E 's|/home/\[USERNAME\]|/[USER_HOME]|g')
    input=$(echo "$input" | sed -E 's/\[EXAMPLE_[A-Z_]+\]/[REDACTED_TOKEN]/g')
    input=$(echo "$input" | sed -E 's|://\[EXAMPLE_[A-Z_]+\]:\[EXAMPLE_[A-Z_]+\]@|://[REDACTED]@|g')
    input=$(echo "$input" | sed -E 's/10\.0\.0\.[0-9]+/[PRIVATE_IP]/g')
    input=$(echo "$input" | sed -E 's/10\.[0-9.]+/[PRIVATE_IP]/g')
    
    echo "$input"
}
```

## Security Review Process

Before creating any PR:

1. **Draft the content** - Write the PR description and comments
2. **Apply automated sanitization** - Run pattern-based replacements
3. **Manual security review** - Read through and check for sensitive info
4. **Verify usefulness** - Ensure sanitized content is still helpful
5. **Create PR** - Only after sanitization is complete

## Incident Response

If sensitive information is accidentally leaked in a PR:

1. **Immediately close the PR** - Don't wait
2. **Rotate compromised credentials** - Assume they're compromised
3. **Delete the PR branch** - Remove from repository
4. **Review sanitization process** - Identify what went wrong
5. **Update patterns** - Add new patterns to prevent recurrence
6. **Document the incident** - Learn from the mistake

## Conclusion

Security sanitization is not optional. Every piece of content that goes into a PR must be sanitized. When in doubt, redact. It's better to provide less information than to leak sensitive data.

**Remember: Once information is in a PR, it's public. You cannot take it back.**