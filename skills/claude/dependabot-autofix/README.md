# Dependabot Auto-Fix Skill (Claude Code)

**Version:** 2.0.0
**Last Updated:** 2026-06-26

A Claude Code skill that autonomously fixes GitHub Dependabot security alerts by updating dependencies, **automatically discovering migration guides, applying code fixes for breaking changes**, verifying builds/tests, and creating pull requests.

**This is an initial version, please use with caution. The code was generated using AI tools.**

---

## Quick Start

### Installation

Copy the skill directory to your Claude Code skills location:

```bash
# From the ai-tools repository root
cp -r skills/claude/dependabot-autofix ~/.claude/skills/dependabot-autofix
```

Or symlink it:

```bash
ln -s "$(pwd)/skills/claude/dependabot-autofix" ~/.claude/skills/dependabot-autofix
```

### Prerequisites

- [GitHub CLI (`gh`)](https://cli.github.com/) installed and authenticated
- Git repository with a GitHub remote
- Dependabot enabled on the repository
- Write access to the repository (or a fork with push access)
- A working build/test setup in the project

### Usage

Invoke the skill from Claude Code:

```
/dependabot-autofix
/dependabot-autofix critical
/dependabot-autofix lodash
/dependabot-autofix top 5
```

**Arguments:**
| Argument | Description |
|----------|-------------|
| *(empty)* or `all` | Fix all open alerts |
| `critical`, `high`, `medium`, `low` | Fix alerts at that severity and above |
| *library name* (e.g., `lodash`) | Fix alerts for a specific library |
| `top N` (e.g., `top 5`) | Fix the N most severe alerts |

---

## What Does This Skill Do?

The Dependabot Auto-Fix skill automates the tedious process of fixing security vulnerabilities in your dependencies. Instead of manually updating each vulnerable package, running tests, and creating pull requests, this skill does it all for you. **The skill intelligently handles breaking changes by discovering migration guides and automatically applying code fixes.**

**The skill:**
1. Fetches open Dependabot security alerts from your GitHub repository
2. Lets you choose which alerts to fix (by library, severity, or count)
3. Updates dependencies to patched versions
4. Runs your test suite to verify nothing breaks
5. Discovers migration guides from 4 tiers of sources when tests fail
6. Automatically applies code fixes for breaking changes (imports, APIs, configs)
7. Retries up to 3 times with different fix strategies
8. Creates pull requests for successful fixes
9. Creates draft PRs with diagnostics when automatic fixes don't resolve all issues

---

## Workflow Overview

The skill executes a 13-phase workflow:

| Phase | Description |
|-------|-------------|
| 1 | Initialize — verify prerequisites (git, gh, package manager) |
| 1.5 | Select remotes — detect alert remote and push remote (fork support) |
| 2 | Fetch alerts — retrieve open Dependabot alerts via GitHub API |
| 3 | Filter & group — organize alerts by package and severity |
| 4 | User selection — present options, apply filters from arguments |
| 5 | Process loop — create branch per library, iterate through selected alerts |
| 6 | Update dependency — update package to patched version |
| 7 | Run tests — verify build and tests pass |
| 8 | Auto-fix — discover migration guides, apply code fixes (up to 3 attempts) |
| 9 | Create PR — push branch and create pull request |
| 10 | Draft PR — create draft PR with diagnostics if fix fails |
| 11 | Cleanup — return to original branch, move to next library |
| 13 | Final report — summary of all fixes attempted |

---

## Fork Workflow Support

The skill supports fork-based workflows where:
- **Alert remote** (e.g., `upstream`) — the repository with Dependabot alerts and PR target
- **Push remote** (e.g., `origin`) — your fork where branches are pushed

When multiple GitHub remotes are detected, the skill asks you to select which remote to use for each purpose.

---

## Supported Ecosystems

| Ecosystem | Package File | Supported |
|-----------|-------------|-----------|
| npm/yarn/pnpm | `package.json` | Yes |
| Python (pip) | `requirements.txt` | Yes |
| Python (Pipenv) | `Pipfile` | Yes |
| Python (Poetry) | `pyproject.toml` | Yes |
| Maven | `pom.xml` | Yes |
| Gradle | `build.gradle` / `build.gradle.kts` | Yes |
| Ruby (Bundler) | `Gemfile` | Yes |
| PHP (Composer) | `composer.json` | Yes |
| Go | `go.mod` | Yes |
| Rust (Cargo) | `Cargo.toml` | Yes |
| .NET (NuGet) | `*.csproj` | Yes |

---

## How Code Fixing Works

When a dependency update causes test failures, the skill follows a multi-step auto-fix process:

### Migration Guide Discovery (4 Tiers)

1. **GitHub Repository** — CHANGELOG.md, UPGRADING.md, release notes
2. **Package Registry** — package metadata, README links
3. **Official Documentation** — docs sites, readthedocs
4. **Community Resources** — Stack Overflow, blog posts, GitHub Issues

### Fix Application Strategy

The skill applies fixes iteratively, prioritizing safer changes:

1. **Attempt 1: Import fixes** — updated import paths, module names
2. **Attempt 2: Configuration fixes** — updated config files, options
3. **Attempt 3: API signature fixes** — updated function calls, parameters

After each attempt, tests are re-run. If tests pass, a PR is created. If all 3 attempts fail, a draft PR is created with diagnostic information.

---

## Security & Privacy

The skill sanitizes all PR content before submission:

- Absolute file paths are converted to relative paths
- Credentials, tokens, and API keys are redacted
- Private IP addresses and internal hostnames are removed
- Environment variable values are redacted (names preserved)
- URLs with embedded credentials are sanitized
- Stack traces with sensitive paths are cleaned

See `security-checklist.md` and `guides/security-sanitization.md` for details.

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `gh` not authenticated | Run `gh auth login` in your terminal |
| No alerts found | Ensure Dependabot is enabled in the repository's Security settings |
| Permission denied on push | Check write access to the push remote, or use a fork workflow |
| Tests timeout | The default timeout is 600 seconds; for longer test suites, the skill may need adjustment |
| Package manager not detected | Ensure your project has a standard package manifest file in the repository root |

---

## Limitations

- Maximum of 3 auto-fix attempts per library
- Test timeout of 600 seconds (10 minutes)
- Requires `gh` CLI for all GitHub API interactions
- Cannot fix alerts that have no patched version available
- Code fixes are best-effort — complex breaking changes may require manual intervention

---

## File Structure

```
skills/claude/dependabot-autofix/
├── SKILL.md                          # Main skill definition
├── README.md                         # This file
├── alert-selection-guide.md          # User selection interaction patterns
├── dependency-update-strategies.md   # Ecosystem-agnostic update strategy
├── security-checklist.md             # Pre-PR security verification checklist
├── test-verification-checklist.md    # Build/test verification checklist
├── pr-template.md                    # PR description templates
└── guides/
    ├── alert-fetching.md             # Remote detection and alert fetching
    ├── build-test-verification.md    # Test command detection and execution
    ├── code-modification.md          # Code fix implementation
    ├── dependency-update.md          # Dependency update strategies
    ├── migration-guide-discovery.md  # 4-tier migration guide search
    ├── pr-creation.md                # PR creation and formatting
    └── security-sanitization.md      # Content sanitization rules
```

---

## See Also

- [Bob version](../../bob/dependabot-autofix/) — the original skill written for the Bob AI agent
- [ai-tools repository](https://github.com/baldimir/ai-tools) — source repository
