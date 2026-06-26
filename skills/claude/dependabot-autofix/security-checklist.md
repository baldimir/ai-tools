# Security Checklist for PR Creation

## Overview

This checklist MUST be completed before creating any Pull Request. Every item must be verified to ensure no sensitive information is leaked.

## Pre-PR Security Checklist

### 1. Content Preparation

- [ ] **Draft PR content created** - All descriptions, comments, and diagnostics drafted
- [ ] **Content reviewed for completeness** - Ensure all necessary information is included
- [ ] **Automated sanitization applied** - Run pattern-based sanitization scripts
- [ ] **Manual review completed** - Human verification of sanitized content

### 2. Credentials and Secrets

- [ ] **No API keys visible** - Check for any API key patterns
- [ ] **No passwords or tokens** - Verify no authentication credentials
- [ ] **No OAuth tokens** - Check for OAuth/JWT tokens
- [ ] **No SSH keys** - Verify no private keys or certificates
- [ ] **No database credentials** - Check for DB usernames/passwords
- [ ] **No service account credentials** - Verify no service account info
- [ ] **No session tokens** - Check for session identifiers

**Example Patterns to Check:**
```
❌ API_KEY=[EXAMPLE_API_KEY]
❌ password: [EXAMPLE_PASSWORD]
❌ Bearer [EXAMPLE_JWT_TOKEN]
✅ API_KEY=[REDACTED]
✅ password: [REDACTED]
✅ Bearer [REDACTED_TOKEN]
```

### 3. File Paths

- [ ] **All paths are relative** - No absolute paths with usernames
- [ ] **No home directory paths** - Check for /Users/ or /home/
- [ ] **No Windows user paths** - Check for C:\Users\
- [ ] **Workspace-relative paths used** - Use ./ prefix for clarity

**Example Patterns to Check:**
```
❌ /Users/[USERNAME]/projects/myapp/src/main.ts
❌ /home/[USERNAME]/workspace/app/config.json
❌ C:\Users\[USERNAME]\Documents\project\file.txt
✅ ./src/main.ts
✅ ./config.json
✅ ./file.txt
```

### 4. Environment Variables

- [ ] **No environment variable values** - Only names are safe
- [ ] **Configuration values redacted** - Check for config values
- [ ] **Runtime parameters sanitized** - Verify no runtime secrets

**Example Patterns to Check:**
```
❌ DATABASE_URL=postgresql://[EXAMPLE_USER]:[EXAMPLE_PASS]@localhost:5432/db
❌ API_KEY=[EXAMPLE_API_KEY]
❌ SECRET_TOKEN=[EXAMPLE_SECRET_TOKEN]
✅ DATABASE_URL=[REDACTED]
✅ API_KEY=[REDACTED]
✅ SECRET_TOKEN=[REDACTED]
```

### 5. URLs and Network Information

- [ ] **No URLs with credentials** - Check for username:password in URLs
- [ ] **No tokens in query parameters** - Verify no ?token= or ?key=
- [ ] **No private IP addresses** - Check for 10.x, 192.168.x, 172.16-31.x
- [ ] **No internal hostnames** - Verify no .internal or .local domains
- [ ] **No MAC addresses** - Check for hardware addresses

**Example Patterns to Check:**
```
❌ https://[EXAMPLE_USER]:[EXAMPLE_PASS]@api.example.com/data
❌ https://api.example.com/data?api_key=[EXAMPLE_API_KEY]
❌ Connecting to 10.0.0.100:5432
❌ Server: database.example.com
❌ MAC: [EXAMPLE_MAC_ADDRESS]
✅ https://[REDACTED]@api.example.com/data
✅ https://api.example.com/data?api_key=[REDACTED]
✅ Connecting to [PRIVATE_IP]:5432
✅ Server: [INTERNAL_HOST]
✅ MAC: [REDACTED_MAC]
```

### 6. Stack Traces and Error Messages

- [ ] **Stack traces sanitized** - All paths converted to relative
- [ ] **Error messages sanitized** - No sensitive values in errors
- [ ] **Debug output cleaned** - No internal debugging information
- [ ] **Line numbers preserved** - Keep useful debugging info

**Example Patterns to Check:**
```
❌ at Database.connect (/Users/[USERNAME]/projects/app/src/db.ts:45:12)
❌ Error: Failed to connect to postgresql://[EXAMPLE_USER]:[EXAMPLE_PASS]@database.example.com:5432
❌ Debug: API_KEY=[EXAMPLE_KEY], USER=[EXAMPLE_USER], PASSWORD=[EXAMPLE_PASS]
✅ at Database.connect (./src/db.ts:45:12)
✅ Error: Failed to connect to database [connection details redacted]
✅ Debug: API_KEY=[REDACTED], USER=[REDACTED], PASSWORD=[REDACTED]
```

### 7. Test Output

- [ ] **Test names safe** - Test names don't contain sensitive info
- [ ] **Test output sanitized** - No sensitive data in test results
- [ ] **Assertion values redacted** - No sensitive values in assertions
- [ ] **Test database info redacted** - No test DB credentials

**Example Patterns to Check:**
```
❌ ✓ should connect to postgresql://[EXAMPLE_USER]:[EXAMPLE_PASS]@10.0.0.100:5432
❌ ✗ should authenticate with API key [EXAMPLE_API_KEY]
❌ ✓ should load config from /Users/[USERNAME]/projects/app/config/test.json
✅ ✓ should connect to database
✅ ✗ should authenticate user (authentication failed)
✅ ✓ should load config from ./config/test.json
```

### 8. Build Logs

- [ ] **Build output sanitized** - No sensitive info in build logs
- [ ] **Compilation paths relative** - All paths are relative
- [ ] **Build server info redacted** - No internal build server details
- [ ] **Artifact paths sanitized** - No absolute artifact paths

**Example Patterns to Check:**
```
❌ Building on ci-server.example.com
❌ Output: /var/jenkins/jobs/myapp/builds/123/artifacts/app.jar
❌ Using credentials from /home/[USERNAME]/.npmrc
✅ Building on CI server
✅ Output: ./build/artifacts/app.jar
✅ Using credentials from configuration
```

### 9. Configuration Files

- [ ] **No config file contents** - Don't include full config files
- [ ] **Config values redacted** - Redact any config values shown
- [ ] **Connection strings redacted** - No database/service connections
- [ ] **Service URLs sanitized** - No internal service URLs

**Example Patterns to Check:**
```
❌ {
     "database": "postgresql://[EXAMPLE_USER]:[EXAMPLE_PASS]@database.example.com:5432/prod",
     "apiKey": "[EXAMPLE_API_KEY]"
   }
❌ redis_url: redis://:[EXAMPLE_PASS]@cache.example.com:6379
✅ {
     "database": "[REDACTED_CONNECTION_STRING]",
     "apiKey": "[REDACTED]"
   }
✅ redis_url: [REDACTED_CONNECTION_STRING]
```

### 10. Git and Version Control

- [ ] **No internal email addresses** - Redact internal emails
- [ ] **Commit messages sanitized** - No sensitive info in commits
- [ ] **Branch names safe** - No sensitive info in branch names
- [ ] **Author information safe** - Public emails only

**Example Patterns to Check:**
```
❌ Author: user@example.com
❌ Commit: "Added API key [EXAMPLE_KEY] for production"
❌ Branch: feature/add-secret-key-[EXAMPLE_KEY]
✅ Author: [REDACTED_EMAIL]
✅ Commit: "Added API key [REDACTED] for production"
✅ Branch: feature/add-authentication
```

## Safe Information Verification

### Confirm These Are Included (Safe)

- [ ] **Dependency names** - Package names are safe (e.g., lodash, react)
- [ ] **Dependency versions** - Version numbers are safe (e.g., 4.17.21)
- [ ] **CVE identifiers** - CVE IDs are safe (e.g., CVE-2021-23337)
- [ ] **Public error types** - Error types without values (e.g., TypeError)
- [ ] **Test names** - Test descriptions without output
- [ ] **Relative file paths** - Paths starting with ./ or ../
- [ ] **Public URLs** - Documentation and package registry URLs
- [ ] **Generic environment names** - production, staging, development

### Examples of Safe Content

```
✅ Updated lodash from 4.17.20 to 4.17.21
✅ Fixes CVE-2021-23337 (Prototype Pollution)
✅ All tests passed (15 passed, 0 failed)
✅ Modified files: ./src/utils.ts, ./package.json
✅ Documentation: https://lodash.com/docs/4.17.21
✅ Deployed to staging environment
✅ Build completed successfully
```

## Final Review Checklist

### Before Creating PR

- [ ] **Read entire PR description** - Review all content
- [ ] **Check all code blocks** - Verify no sensitive info in code examples
- [ ] **Review all links** - Ensure no URLs with credentials
- [ ] **Verify all paths** - Confirm all paths are relative
- [ ] **Check all examples** - Ensure examples don't leak info
- [ ] **Review diagnostics** - Sanitize all diagnostic information
- [ ] **Verify test results** - Ensure test output is sanitized
- [ ] **Check error messages** - Confirm errors are sanitized

### Content Quality Check

- [ ] **Information is useful** - Sanitized content still provides value
- [ ] **Context is clear** - Readers can understand the changes
- [ ] **Debugging info preserved** - Useful debugging info retained (line numbers, file names)
- [ ] **No over-redaction** - Haven't removed too much useful information

### Security Verification

- [ ] **No credentials visible** - Triple-check for any credentials
- [ ] **No internal info visible** - No internal network/system info
- [ ] **No personal info visible** - No usernames, emails, etc.
- [ ] **Safe to publish** - Content is safe for public viewing

## Quick Reference: Common Mistakes

### ❌ Common Mistakes to Avoid

1. **Including full error messages with credentials**
   ```
   Error: Authentication failed with key [EXAMPLE_API_KEY]
   ```

2. **Showing absolute paths**
   ```
   Modified: /Users/[USERNAME]/projects/myapp/src/main.ts
   ```

3. **Including environment variable values**
   ```
   Using DATABASE_URL=postgresql://[EXAMPLE_USER]:[EXAMPLE_PASS]@localhost:5432/db
   ```

4. **Showing URLs with tokens**
   ```
   Fetching from https://api.example.com/data?token=[EXAMPLE_TOKEN]
   ```

5. **Including private IPs**
   ```
   Connected to database at 10.0.0.100:5432
   ```

6. **Showing internal hostnames**
   ```
   Deployed to server.example.com
   ```

7. **Including full stack traces with absolute paths**
   ```
   at main (/home/[USERNAME]/workspace/app/src/index.ts:10:5)
   ```

8. **Showing configuration file contents**
   ```
   Config: { apiKey: "[EXAMPLE_API_KEY]", secret: "[EXAMPLE_SECRET]" }
   ```

### ✅ Correct Approach

1. **Redact credentials in error messages**
   ```
   Error: Authentication failed [credentials redacted]
   ```

2. **Use relative paths**
   ```
   Modified: ./src/main.ts
   ```

3. **Redact environment variable values**
   ```
   Using DATABASE_URL=[REDACTED]
   ```

4. **Redact tokens in URLs**
   ```
   Fetching from https://api.example.com/data?token=[REDACTED]
   ```

5. **Redact private IPs**
   ```
   Connected to database at [PRIVATE_IP]:5432
   ```

6. **Redact internal hostnames**
   ```
   Deployed to [INTERNAL_HOST]
   ```

7. **Sanitize stack traces**
   ```
   at main (./src/index.ts:10:5)
   ```

8. **Redact configuration values**
   ```
   Config: { apiKey: "[REDACTED]", secret: "[REDACTED]" }
   ```

## Emergency Procedures

### If Sensitive Information Is Accidentally Leaked

1. **IMMEDIATELY close the PR** - Don't wait for review
2. **Rotate all compromised credentials** - Assume they're compromised
3. **Delete the PR branch** - Remove from repository history
4. **Notify security team** - If applicable
5. **Review this checklist** - Identify what was missed
6. **Update sanitization patterns** - Prevent future occurrences
7. **Document the incident** - Learn from the mistake

## Sign-Off

Before creating the PR, confirm:

- [ ] **I have completed all items in this checklist**
- [ ] **I have manually reviewed all content**
- [ ] **I am confident no sensitive information is included**
- [ ] **The PR content is safe for public viewing**

**Date:** _______________

**Reviewer:** _______________

---

## Notes

Use this space to document any special considerations or exceptions:

```
[Add notes here]
```

---

**Remember:** Once information is in a PR, it's public. You cannot take it back. When in doubt, redact.