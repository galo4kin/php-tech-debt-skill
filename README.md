# PHP Tech Debt Skill

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

Claude Code skill for comprehensive PHP technical debt auditing.

## What It Does

PHP Tech Debt Skill combines two complementary approaches into a single audit:

**Tool-based scoring** runs up to 11 static analysis and quality tools (PHPStan, semgrep, phpcs, PHPUnit, and others) and produces a numeric health score on a 100-point scale across 5 categories: Security, Static Analysis, Dependencies, Code Quality, and Test Coverage. Each category is worth 20 points. If some tools are missing, the score normalizes automatically so the total is always out of 100.

**Deep architectural audit** performs a Claude-driven analysis across 9 dimensions (architectural decay, consistency rot, type debt, test debt, dependency debt, performance, error handling, security hygiene, and documentation drift). Every finding includes a `file:line` citation pointing to the exact location in the codebase. No vague claims, no generic advice.

The result is a living `TECH_DEBT_AUDIT.md` document that tracks findings over time. Run the audit again and it shows what has been resolved, what is new, and how the score has changed since the last run.

## Quick Install

All three methods place the skill where Claude Code can discover it. No other configuration is needed.

**Method 1 -- Per-project (recommended):**

```bash
mkdir -p .claude/skills
git clone https://github.com/galo4kin/php-tech-debt-skill.git .claude/skills/php-tech-debt-audit
```

**Method 2 -- Global (all projects):**

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/galo4kin/php-tech-debt-skill.git ~/.claude/skills/php-tech-debt-audit
```

**Method 3 -- curl (single file):**

```bash
mkdir -p .claude/skills/php-tech-debt-audit
curl -o .claude/skills/php-tech-debt-audit/SKILL.md \
  https://raw.githubusercontent.com/galo4kin/php-tech-debt-skill/main/.claude/skills/php-tech-debt-audit/SKILL.md
curl -o .claude/skills/php-tech-debt-audit/report-template.md \
  https://raw.githubusercontent.com/galo4kin/php-tech-debt-skill/main/.claude/skills/php-tech-debt-audit/report-template.md
```

## Usage

Audit the entire project:

```
/php-tech-debt-audit
```

Audit a specific directory:

```
/php-tech-debt-audit src/
```

The skill will detect available tools, run them, perform architectural analysis, and write `TECH_DEBT_AUDIT.md` to the project root.

## Supported Tools

Every tool is optional. The skill uses whatever is available and adjusts scoring accordingly.

| Tool | What It Checks | Install Command | Required |
|------|---------------|-----------------|----------|
| composer audit | CVE vulnerabilities in dependencies | Built into Composer | No |
| PHPStan | Static analysis, type safety, code correctness | `composer require --dev phpstan/phpstan` | No |
| semgrep | Security patterns, injection vulnerabilities, secrets | `pip install semgrep` | No |
| phpcs | Coding standards compliance (PSR-12, custom rulesets) | `composer require --dev squizlabs/php_codesniffer` | No |
| php-cs-fixer | Coding standards (alternative to phpcs) | `composer require --dev friendsofphp/php-cs-fixer` | No |
| PHPUnit | Test coverage measurement | `composer require --dev phpunit/phpunit` | No |
| Codeception | Test coverage (alternative to PHPUnit) | `composer require --dev codeception/codeception` | No |
| phpmd | Cyclomatic complexity, design rules, unused code | `composer require --dev phpmd/phpmd` | No |
| phpcpd | Copy/paste detection across codebase | `composer require --dev sebastian/phpcpd` | No |
| phploc | Codebase size statistics (LOC, classes, methods) | `composer require --dev phploc/phploc` | No |
| pdepend | Dependency metrics and software quality metrics | `composer require --dev pdepend/pdepend` | No |

The skill detects each tool by checking global binaries (`which`), vendor binaries (`vendor/bin/`), `composer.json` require-dev entries, and associated config files. Missing tools are listed with install commands in both the console output and the final report.

## Docker Support

The skill auto-detects containerized PHP environments. If PHP is not available on the host machine, it searches for `docker-compose.yml` (in the project root and `docker/` directory), identifies the PHP service, verifies the container is running, and runs all tools through `docker compose exec -T <service>`. No configuration is needed -- the detection is fully automatic.

When running in Docker mode, the skill prefixes all PHP and Composer commands with the appropriate `docker compose exec` call. Tools that read source files directly (like semgrep) continue to run on the host.

If neither host PHP nor Docker PHP is found, the skill skips tool-based scoring entirely and performs Claude-driven architectural analysis using file reading only.

## Scoring

The health score uses a 5-category system, each worth 20 points:

| Category | What It Measures | Tools Used |
|----------|-----------------|------------|
| **Security** (0-20) | CVE vulnerabilities in dependencies, injection patterns, hardcoded secrets, dangerous function usage | composer audit, semgrep |
| **Static Analysis** (0-20) | Type errors, code correctness, complexity violations, design rule violations, unused code | PHPStan, phpmd |
| **Dependencies** (0-20) | Outdated packages, major version gaps, abandoned packages, composer.lock freshness | composer outdated |
| **Code Quality** (0-20) | Coding standard violations, code duplication, class and method size | phpcs (or php-cs-fixer), phpcpd, phploc |
| **Test Coverage** (0-20) | Line coverage percentage from automated test suites | PHPUnit (or Codeception) + Xdebug/PCOV |

**Normalization:** If only 3 out of 5 categories have tools available, the score still reports out of 100. Each available category receives a proportionally higher weight so that projects with fewer tools are not penalized unfairly. The report shows both raw category scores (X/20) and the normalized total.

## Architectural Audit

The 9-dimension architectural audit examines the codebase through Claude-driven analysis with targeted searches. Every finding requires a `file:line` citation.

1. **Architectural decay** -- God classes, service locator abuse, circular dependencies, dead code, controllers with business logic
2. **Consistency rot** -- Multiple patterns for the same task, mixed autoloading, inconsistent naming conventions, conflicting config formats
3. **Type and contract debt** -- Missing `declare(strict_types=1)`, missing return types, `mixed` parameters without validation, untyped properties
4. **Test debt** -- Coverage gaps on critical paths, tests asserting implementation over behavior, skipped tests, tests without assertions
5. **Dependency debt** -- CVEs, dev packages in production require, unused packages, unmaintained dependencies, PHP version constraint issues
6. **Performance** -- N+1 queries, `file_get_contents()` for HTTP calls, missing database indexes, synchronous I/O in request paths
7. **Error handling** -- Empty catch blocks, catches without logging or re-throw, overly broad exception catches, `@` error suppression
8. **Security hygiene** -- Direct superglobal access, SQL string concatenation, `eval()`/`exec()`/`system()`, hardcoded credentials, weak password hashing
9. **Documentation drift** -- `@param` types not matching signatures, public methods without docblocks, stale TODO/FIXME comments older than 6 months

## Repeat Runs

Running `/php-tech-debt-audit` on a project that already has a `TECH_DEBT_AUDIT.md` triggers diff-aware tracking:

- **ACTIVE** -- A previously reported finding is still present at the same location.
- **RESOLVED** -- A previously reported finding is no longer detected. It moves to a collapsed "Previously Resolved" section with the resolution date.
- **SHIFTED** -- The same issue exists but the line number changed (code was added or removed above it). The line reference updates, the finding ID stays the same.
- **NEW** -- An issue found in the current run that was not present in the previous run. It gets the next available finding ID.

The score line shows the delta: `72/100 (+8 since 2025-03-15)`. Each run appends to the Run History table in the report, building a trend over time. A YAML metrics baseline block at the bottom of the report stores raw values for programmatic comparison.

## Example Output

After the audit completes, the skill prints a console summary:

```
PHP Tech Debt Audit Complete
Health Score: 72/100
  Security:        18/20 ✓
  Static Analysis: 14/20 ⚠
  Dependencies:    12/20 ⚠
  Code Quality:    16/20 ✓
  Test Coverage:   12/20 ⚠

Findings: 47 total (2 critical, 8 high, 22 medium, 15 low)
Quick Wins: 6 items
Report: TECH_DEBT_AUDIT.md
```

The full report is written to `TECH_DEBT_AUDIT.md` and includes an executive summary, detailed tool results, an architectural findings table with file:line citations and severity/effort estimates, prioritized recommendations, quick wins, and appendices with tool versions and metrics baselines.

## Contributing

Contributions are welcome.

1. Fork the repository.
2. Create a feature branch: `git checkout -b my-feature`.
3. Make your changes and commit them.
4. Push to your fork: `git push origin my-feature`.
5. Open a pull request against `main`.

Please keep changes focused and include a clear description of what the change does and why.

## License

[MIT](LICENSE)
