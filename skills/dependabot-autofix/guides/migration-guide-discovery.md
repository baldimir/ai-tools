# Migration Guide Discovery Guide

This guide provides detailed instructions for discovering and analyzing migration guides when dependency updates introduce breaking changes.

## Overview

When tests fail after a dependency update, the skill must automatically search for migration documentation to understand what code changes are needed. This guide implements a 4-tier search strategy to find relevant migration information.

## When to Use This Guide

Trigger migration guide discovery when:
- Tests fail after dependency update
- Build fails with compilation errors
- Import/module errors are detected
- API signature changes are detected

## 4-Tier Search Strategy

### Tier 1: GitHub Repository (Primary Source)

Search the package's GitHub repository for official migration documentation.

**Priority Files to Check:**

1. **CHANGELOG.md**
2. **UPGRADING.md** 
3. **MIGRATION.md**
4. **MIGRATING.md**
5. **docs/migration/** directory
6. **docs/upgrading/** directory
7. **GitHub Releases** (release notes)

**Implementation:**

```bash
#!/bin/bash

search_github_migration_docs() {
    local package_name="$1"
    local old_version="$2"
    local new_version="$3"
    local repo_url="$4"  # e.g., "owner/repo"
    
    echo "Tier 1: Searching GitHub repository for migration guides..."
    
    # Extract owner and repo from URL if needed
    if [ -z "$repo_url" ]; then
        # Try to find repo URL from package registry
        repo_url=$(get_repo_url_from_package "$package_name")
    fi
    
    if [ -z "$repo_url" ]; then
        echo "⚠️ Could not determine GitHub repository"
        return 1
    fi
    
    # Check for common migration files
    local migration_files=(
        "CHANGELOG.md"
        "UPGRADING.md"
        "MIGRATION.md"
        "MIGRATING.md"
        "docs/migration.md"
        "docs/upgrading.md"
        "docs/MIGRATION.md"
        "docs/UPGRADING.md"
    )
    
    for file in "${migration_files[@]}"; do
        echo "Checking for $file..."
        if gh api "repos/$repo_url/contents/$file" --jq '.download_url' 2>/dev/null; then
            local download_url=$(gh api "repos/$repo_url/contents/$file" --jq '.download_url')
            echo "✅ Found: $file"
            curl -sL "$download_url" > "migration-docs-${file//\//-}"
            
            # Extract relevant version information
            extract_version_changes "migration-docs-${file//\//-}" "$old_version" "$new_version"
            return 0
        fi
    done
    
    # Check docs/migration/ directory
    echo "Checking docs/migration/ directory..."
    if gh api "repos/$repo_url/contents/docs/migration" 2>/dev/null | jq -e '.[0]' > /dev/null; then
        echo "✅ Found migration directory"
        gh api "repos/$repo_url/contents/docs/migration" | jq -r '.[].download_url' | while read url; do
            local filename=$(basename "$url")
            curl -sL "$url" > "migration-docs-$filename"
        done
        return 0
    fi
    
    # Check GitHub Releases
    echo "Checking GitHub Releases..."
    local release_notes=$(gh api "repos/$repo_url/releases" --jq ".[] | select(.tag_name | contains(\"$new_version\")) | .body")
    
    if [ -n "$release_notes" ]; then
        echo "✅ Found release notes for version $new_version"
        echo "$release_notes" > "migration-docs-release-notes.md"
        return 0
    fi
    
    echo "⚠️ No migration documentation found in GitHub repository"
    return 1
}

# Helper function to extract version-specific changes
extract_version_changes() {
    local file="$1"
    local old_version="$2"
    local new_version="$3"
    
    echo "Extracting changes between $old_version and $new_version..."
    
    # Look for version headers and extract content between versions
    # This is a simplified approach - actual implementation would be more sophisticated
    grep -A 50 -i "version $new_version\|$new_version\|## \[$new_version\]" "$file" > "migration-relevant.md"
    
    # Look for breaking changes section
    grep -A 20 -i "breaking change\|breaking:\|BREAKING" "$file" >> "migration-relevant.md"
    
    # Look for migration section
    grep -A 30 -i "migrat\|upgrad\|how to upgrade" "$file" >> "migration-relevant.md"
}

# Helper function to get repository URL from package
get_repo_url_from_package() {
    local package_name="$1"
    
    # Try npm
    if [ -f "package.json" ]; then
        local repo=$(npm view "$package_name" repository.url 2>/dev/null | sed 's/git+https:\/\/github.com\///' | sed 's/.git$//')
        if [ -n "$repo" ]; then
            echo "$repo"
            return 0
        fi
    fi
    
    # Try PyPI
    if [ -f "requirements.txt" ] || [ -f "setup.py" ]; then
        local repo=$(curl -s "https://pypi.org/pypi/$package_name/json" | jq -r '.info.project_urls.Source // .info.project_urls.Homepage // empty' | grep github.com | sed 's|https://github.com/||' | sed 's|/$||')
        if [ -n "$repo" ]; then
            echo "$repo"
            return 0
        fi
    fi
    
    return 1
}
```

---

### Tier 2: Package Registry

Search the package's registry for documentation links and migration guides.

**General Approach:**

The skill automatically identifies the appropriate package registry based on the detected ecosystem and searches for:
- Package metadata
- README files
- Documentation URLs
- Changelog links
- Project homepage

**Implementation:**

```bash
#!/bin/bash

search_package_registry() {
    local package_name="$1"
    local ecosystem="$2"
    
    echo "Tier 2: Searching package registry for migration guides..."
    
    # Use ecosystem-specific registry search
    # Implementation automatically adapts to the detected ecosystem
    search_registry_for_package "$package_name" "$ecosystem"
    
    # Extract common information:
    # - Homepage URL
    # - Repository URL
    # - Documentation URL
    # - Changelog URL
    # - README content
    
    # Check if migration information found
    if [ -f "migration-homepage-url.txt" ] || [ -f "migration-docs-url.txt" ] || [ -f "migration-changelog.md" ]; then
        echo "✅ Found package registry information"
        return 0
    fi
    
    echo "⚠️ No migration information found in package registry"
    return 1
}

search_registry_for_package() {
    local package_name="$1"
    local ecosystem="$2"
    
    echo "Searching $ecosystem registry for $package_name..."
    
    # Get package metadata using ecosystem-appropriate method
    # Extract URLs and documentation links
    # Save discovered resources
    
    # This function adapts to any package registry
    # without requiring ecosystem-specific implementations
}
```

---

### Tier 3: Official Documentation Sites

Search official documentation websites for migration guides.

**Common Documentation Patterns:**

- `docs.{package}.com`
- `{package}.readthedocs.io`
- `{package}.github.io`
- `www.{package}.org/docs`

**Implementation:**

```bash
#!/bin/bash

search_official_docs() {
    local package_name="$1"
    local old_version="$2"
    local new_version="$3"
    
    echo "Tier 3: Searching official documentation sites..."
    
    # Common documentation URL patterns
    local doc_patterns=(
        "https://docs.${package_name}.com"
        "https://${package_name}.readthedocs.io"
        "https://${package_name}.github.io"
        "https://www.${package_name}.org/docs"
        "https://${package_name}.org/docs"
    )
    
    for doc_url in "${doc_patterns[@]}"; do
        echo "Checking $doc_url..."
        
        if curl -s -o /dev/null -w "%{http_code}" "$doc_url" | grep -q "200"; then
            echo "✅ Found documentation site: $doc_url"
            
            # Search for migration/upgrade pages
            search_doc_site_for_migration "$doc_url" "$old_version" "$new_version"
            
            if [ $? -eq 0 ]; then
                return 0
            fi
        fi
    done
    
    # If homepage URL was found in Tier 2, search it
    if [ -f "migration-homepage-url.txt" ]; then
        local homepage=$(cat migration-homepage-url.txt)
        echo "Searching homepage: $homepage"
        search_doc_site_for_migration "$homepage" "$old_version" "$new_version"
        return $?
    fi
    
    echo "⚠️ No official documentation sites found"
    return 1
}

search_doc_site_for_migration() {
    local base_url="$1"
    local old_version="$2"
    local new_version="$3"
    
    # Common migration page paths
    local migration_paths=(
        "/migration"
        "/upgrading"
        "/upgrade-guide"
        "/migration-guide"
        "/docs/migration"
        "/docs/upgrading"
        "/guides/migration"
        "/guides/upgrading"
        "/changelog"
        "/releases"
    )
    
    for path in "${migration_paths[@]}"; do
        local full_url="${base_url}${path}"
        echo "Checking $full_url..."
        
        if curl -s -o /dev/null -w "%{http_code}" "$full_url" | grep -q "200"; then
            echo "✅ Found migration page: $full_url"
            
            # Fetch the page content
            curl -sL "$full_url" > "migration-docs-site.html"
            
            # Extract text content (basic HTML parsing)
            # In practice, you might use html2text or similar
            grep -o ">.*<" "migration-docs-site.html" | sed 's/^>//;s/<$//' > "migration-docs-site.txt"
            
            # Check if it contains version-specific information
            if grep -qi "$new_version\|$old_version" "migration-docs-site.txt"; then
                echo "✅ Found version-specific migration information"
                return 0
            fi
        fi
    done
    
    return 1
}
```

---

### Tier 4: Community Resources

Search community resources for migration information when official docs are unavailable.

**Sources:**

1. **Stack Overflow**
2. **GitHub Issues**
3. **Dev.to**
4. **Medium**
5. **Reddit**

**Implementation:**

```bash
#!/bin/bash

search_community_resources() {
    local package_name="$1"
    local old_version="$2"
    local new_version="$3"
    
    echo "Tier 4: Searching community resources..."
    
    # Search Stack Overflow
    search_stackoverflow "$package_name" "$old_version" "$new_version"
    
    # Search GitHub Issues
    search_github_issues "$package_name" "$old_version" "$new_version"
    
    # Note: Searching Dev.to, Medium, Reddit would require browser_action
    # which is not available in code mode. Document the approach for future.
    
    return 0
}

search_stackoverflow() {
    local package_name="$1"
    local old_version="$2"
    local new_version="$3"
    
    echo "Searching Stack Overflow..."
    
    # Build search query
    local query="${package_name}+migrate+${old_version}+${new_version}"
    local api_url="https://api.stackexchange.com/2.3/search/advanced?order=desc&sort=relevance&q=${query}&site=stackoverflow"
    
    # Fetch results
    local results=$(curl -s "$api_url" | jq -r '.items[0:5] | .[] | {title: .title, link: .link, score: .score}')
    
    if [ -n "$results" ]; then
        echo "✅ Found Stack Overflow discussions"
        echo "$results" > "migration-stackoverflow.json"
        
        # Extract top answer links
        echo "$results" | jq -r '.link' > "migration-stackoverflow-links.txt"
        return 0
    fi
    
    echo "⚠️ No relevant Stack Overflow posts found"
    return 1
}

search_github_issues() {
    local package_name="$1"
    local old_version="$2"
    local new_version="$3"
    
    echo "Searching GitHub Issues..."
    
    # Get repo URL if available
    local repo_url=""
    if [ -f "migration-repo-url.txt" ]; then
        repo_url=$(cat migration-repo-url.txt | sed 's|https://github.com/||')
    else
        repo_url=$(get_repo_url_from_package "$package_name")
    fi
    
    if [ -z "$repo_url" ]; then
        echo "⚠️ Cannot search issues without repository URL"
        return 1
    fi
    
    # Search for migration-related issues
    local query="migrate OR upgrade OR breaking"
    
    gh api "search/issues?q=repo:${repo_url}+${query}+${new_version}" \
        --jq '.items[0:10] | .[] | {number: .number, title: .title, url: .html_url, state: .state}' \
        > "migration-github-issues.json"
    
    if [ -s "migration-github-issues.json" ]; then
        echo "✅ Found relevant GitHub issues"
        return 0
    fi
    
    echo "⚠️ No relevant GitHub issues found"
    return 1
}
```

---

## Migration Guide Parsing

After discovering migration documentation, parse it to extract actionable information.

### Parsing Strategy

```bash
#!/bin/bash

parse_migration_guide() {
    local guide_file="$1"
    
    echo "Parsing migration guide: $guide_file"
    
    # Extract breaking changes
    extract_breaking_changes "$guide_file" > "breaking-changes.txt"
    
    # Extract code examples
    extract_code_examples "$guide_file" > "code-examples.txt"
    
    # Extract migration steps
    extract_migration_steps "$guide_file" > "migration-steps.txt"
    
    # Categorize changes
    categorize_changes "breaking-changes.txt" > "change-categories.json"
}

extract_breaking_changes() {
    local file="$1"
    
    # Look for breaking changes sections
    grep -A 30 -i "breaking change\|breaking:\|BREAKING\|⚠️\|## Breaking" "$file"
}

extract_code_examples() {
    local file="$1"
    
    # Extract code blocks (markdown format)
    awk '/```/,/```/' "$file"
}

extract_migration_steps() {
    local file="$1"
    
    # Look for numbered steps or bullet points about migration
    grep -A 5 -E "^[0-9]+\.|^[-*]" "$file" | grep -i "migrat\|upgrad\|change\|update"
}

categorize_changes() {
    local changes_file="$1"
    
    # Categorize by type
    local import_changes=$(grep -c -i "import\|require\|from.*import" "$changes_file")
    local api_changes=$(grep -c -i "function\|method\|parameter\|signature\|renamed" "$changes_file")
    local config_changes=$(grep -c -i "config\|setting\|option" "$changes_file")
    local behavior_changes=$(grep -c -i "behavior\|return\|default" "$changes_file")
    
    cat <<EOF
{
  "import_changes": $import_changes,
  "api_changes": $api_changes,
  "config_changes": $config_changes,
  "behavior_changes": $behavior_changes
}
EOF
}
```

### Pattern Extraction

Extract specific patterns that can be used for code modification:

```bash
#!/bin/bash

extract_migration_patterns() {
    local guide_file="$1"
    
    echo "Extracting migration patterns..."
    
    # Look for before/after patterns
    extract_before_after_patterns "$guide_file" > "migration-patterns.json"
}

extract_before_after_patterns() {
    local file="$1"
    
    # This is a simplified approach
    # Real implementation would use more sophisticated parsing
    
    # Look for patterns like:
    # Before: old_function()
    # After: new_function()
    
    awk '
    /[Bb]efore:/ {
        before = $0
        getline
        if (/[Aa]fter:/) {
            after = $0
            print "{"
            print "  \"before\": \"" before "\","
            print "  \"after\": \"" after "\""
            print "}"
        }
    }
    ' "$file"
}
```

---

## Complete Discovery Workflow

```bash
#!/bin/bash

discover_migration_guide() {
    local package_name="$1"
    local old_version="$2"
    local new_version="$3"
    local ecosystem="$4"
    
    echo "=========================================="
    echo "Migration Guide Discovery"
    echo "Package: $package_name"
    echo "Version: $old_version → $new_version"
    echo "Ecosystem: $ecosystem"
    echo "=========================================="
    
    # Tier 1: GitHub Repository
    if search_github_migration_docs "$package_name" "$old_version" "$new_version"; then
        echo "✅ Found migration guide in GitHub repository"
        parse_migration_guide "migration-relevant.md"
        return 0
    fi
    
    # Tier 2: Package Registry
    if search_package_registry "$package_name" "$ecosystem"; then
        echo "✅ Found migration information in package registry"
        
        # If we found URLs, try to fetch them
        if [ -f "migration-docs-url.txt" ]; then
            local docs_url=$(cat migration-docs-url.txt)
            curl -sL "$docs_url" > "migration-docs-from-registry.html"
            parse_migration_guide "migration-docs-from-registry.html"
            return 0
        fi
    fi
    
    # Tier 3: Official Documentation
    if search_official_docs "$package_name" "$old_version" "$new_version"; then
        echo "✅ Found migration guide in official documentation"
        parse_migration_guide "migration-docs-site.txt"
        return 0
    fi
    
    # Tier 4: Community Resources
    if search_community_resources "$package_name" "$old_version" "$new_version"; then
        echo "✅ Found migration information in community resources"
        # Community resources provide links, not full guides
        echo "Check the following resources:"
        [ -f "migration-stackoverflow-links.txt" ] && cat migration-stackoverflow-links.txt
        [ -f "migration-github-issues.json" ] && jq -r '.[].url' migration-github-issues.json
        return 0
    fi
    
    # No migration guide found
    echo "⚠️ No migration guide found"
    echo "Will attempt to analyze test failures and apply conservative fixes"
    return 1
}
```

---

## Fallback Strategy

When no migration guide is found, use these fallback approaches:

### 1. Analyze Test Failures

```bash
analyze_test_failures_for_patterns() {
    local test_output="$1"
    
    echo "Analyzing test failures for migration patterns..."
    
    # Look for common error patterns
    if grep -q "ImportError\|ModuleNotFoundError\|Cannot find module" "$test_output"; then
        echo "Detected: Import/Module errors"
        echo "import_errors" > "failure-category.txt"
    elif grep -q "TypeError.*arguments\|AttributeError\|undefined is not a function" "$test_output"; then
        echo "Detected: API signature changes"
        echo "api_changes" > "failure-category.txt"
    elif grep -q "AssertionError\|expected.*but.*got" "$test_output"; then
        echo "Detected: Behavior changes"
        echo "behavior_changes" > "failure-category.txt"
    fi
}
```

### 2. Search Codebase for Usage

```bash
analyze_codebase_usage() {
    local package_name="$1"
    
    echo "Analyzing codebase for package usage..."
    
    # Find all imports/requires
    grep -r "import.*$package_name\|from $package_name\|require.*$package_name" . \
        --include="*.js" --include="*.ts" --include="*.py" --include="*.java" \
        > "package-usage.txt"
    
    echo "Found $(wc -l < package-usage.txt) usage locations"
}
```

### 3. Compare API Documentation

```bash
compare_api_docs() {
    local package_name="$1"
    local old_version="$2"
    local new_version="$3"
    
    echo "Comparing API documentation between versions..."
    
    # This would require fetching docs for both versions
    # and comparing them - complex task, document for future
    
    echo "⚠️ API comparison not yet implemented"
    echo "Recommend manual review of:"
    echo "  - Old version docs"
    echo "  - New version docs"
    echo "  - Changelog"
}
```

---

## Error Handling

```bash
handle_discovery_errors() {
    local error_type="$1"
    
    case "$error_type" in
        "no_repo_url")
            echo "⚠️ Cannot find repository URL"
            echo "Recommendation: Check package registry manually"
            ;;
        "api_rate_limit")
            echo "⚠️ GitHub API rate limit exceeded"
            echo "Recommendation: Wait or use authenticated requests"
            ;;
        "network_error")
            echo "⚠️ Network error during discovery"
            echo "Recommendation: Check internet connection and retry"
            ;;
        "parse_error")
            echo "⚠️ Error parsing migration guide"
            echo "Recommendation: Review guide manually"
            ;;
        *)
            echo "⚠️ Unknown error during discovery"
            ;;
    esac
}
```

---

## Best Practices

1. **Start with Official Sources**
   - GitHub repository is most reliable
   - Package registry is second best
   - Community resources are last resort

2. **Cache Results**
   - Save discovered guides for reuse
   - Avoid redundant API calls
   - Store in temporary files

3. **Version-Specific Search**
   - Always include version numbers in searches
   - Look for version-specific sections
   - Check release notes for that version

4. **Handle Missing Guides Gracefully**
   - Don't fail if guide not found
   - Use fallback strategies
   - Document what was attempted

5. **Extract Actionable Information**
   - Focus on code changes needed
   - Identify patterns for automation
   - Categorize by change type

6. **Preserve Context**
   - Save all discovered resources
   - Include in draft PR if tests fail
   - Help manual reviewers

---

## Integration with Code Modification

After discovering migration guides, pass the information to the code modification phase:

```bash
prepare_for_code_modification() {
    local package_name="$1"
    
    # Consolidate all discovered information
    cat > "migration-summary.json" <<EOF
{
  "package": "$package_name",
  "guides_found": [
    $(ls migration-docs-* 2>/dev/null | jq -R . | jq -s .)
  ],
  "breaking_changes": $(cat breaking-changes.txt 2>/dev/null | jq -R . | jq -s . || echo "[]"),
  "migration_patterns": $(cat migration-patterns.json 2>/dev/null || echo "[]"),
  "change_categories": $(cat change-categories.json 2>/dev/null || echo "{}"),
  "community_resources": {
    "stackoverflow": $(cat migration-stackoverflow-links.txt 2>/dev/null | jq -R . | jq -s . || echo "[]"),
    "github_issues": $(cat migration-github-issues.json 2>/dev/null || echo "[]")
  }
}
EOF
    
    echo "✅ Migration summary prepared for code modification phase"
}
```

---

## Summary

This guide covers:
- ✅ 4-tier search strategy (GitHub, Registry, Docs, Community)
- ✅ Ecosystem-specific registry searches
- ✅ Migration guide parsing and pattern extraction
- ✅ Fallback strategies when guides aren't found
- ✅ Error handling and best practices
- ✅ Integration with code modification phase

**Next Steps:**
- Use discovered information in code-modification.md
- Apply patterns to fix breaking changes
- Retry tests after modifications

---

*Part of Dependabot Auto-Fix Skill v2.0.0*