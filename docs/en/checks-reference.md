---
title: Checks Reference
layout: default
parent: English
nav_order: 3
lang: en
counterpart: /php-tech-debt-skill/ru/checks-reference/
---

{%- include lang-switcher.html -%}

# Checks Reference

The skill performs two types of analysis: tool-based scoring (automated, numeric) and architectural audit (Claude-driven, pattern-based). This page documents every check.

---

## Part 1: Tool-Based Scoring

5 categories, each worth 0-20 points, for a total of 100. If some categories have no tools available, the score normalizes so the total is still out of 100.

### Normalization

```
available_categories = count of categories where at least 1 tool is available
weight_per_category = 100 / available_categories
weighted_score = (raw_score / 20) * weight_per_category
final_score = sum of all weighted_scores, rounded to nearest integer
```

**Example:** If only 3 categories have tools (Security, Static Analysis, Dependencies) with raw scores 18, 12, and 15:

```
weight = 100 / 3 = 33.33
Security:        (18/20) * 33.33 = 30.0
Static Analysis: (12/20) * 33.33 = 20.0
Dependencies:    (15/20) * 33.33 = 25.0
Total: 75/100
```

---

### Category 1: Security (0-20)

**Tools:** composer audit, semgrep

| Score | Criteria |
|-------|----------|
| 20 | Zero CVEs from composer audit AND zero security findings from semgrep |
| 15 | Only low-severity CVEs, no critical/high semgrep findings |
| 10 | Medium-severity CVEs OR moderate semgrep findings |
| 5 | High-severity CVEs present |
| 0 | Critical CVEs OR critical semgrep findings |

If only one tool is available, the score is based on that tool alone (full 20 points still possible).

**Commands run:**
- `composer audit --format=json`
- `semgrep scan --config auto --lang php --json --quiet`

---

### Category 2: Static Analysis (0-20)

**Tools:** PHPStan, phpmd

When **both** tools are available, points split: PHPStan gets 12, phpmd gets 8.

**PHPStan portion (0-12):**

| Score | Criteria |
|-------|----------|
| 12 | Level 6+, fewer than 10 errors |
| 9 | Level 4-5, fewer than 50 errors |
| 6 | Level 1-3 OR more than 100 errors |
| 3 | Level 0 OR more than 500 errors |

**phpmd portion (0-8):**

| Score | Criteria |
|-------|----------|
| 8 | 0 violations |
| 6 | Fewer than 20 violations |
| 4 | Fewer than 100 violations |
| 2 | 100 or more violations |

When only **one** tool is available, it gets the full 0-20 range (thresholds scale proportionally).

**Commands run:**
- `phpstan analyse --error-format=json`
- `phpmd <source-dirs> json cleancode,codesize,controversial,design,naming,unusedcode`

Source directories are detected from PSR-4 autoload mapping in `composer.json`. Fallback: `src/,app/,lib/`.

PHPStan level is read from `phpstan.neon`, `phpstan.neon.dist`, or `phpstan.dist.neon`. If no config exists, level 0 is assumed.

---

### Category 3: Dependencies (0-20)

**Tools:** composer outdated

| Score | Criteria |
|-------|----------|
| 20 | Less than 10% outdated, no major version gaps |
| 15 | Less than 25% outdated OR 1-2 major version gaps |
| 10 | Less than 50% outdated OR 3+ major version gaps |
| 5 | More than 50% outdated |
| 0 | More than 75% outdated OR abandoned packages detected |

Also checks `composer.lock` age via `git log -1 --format=%ci composer.lock`. A lock file not updated in over 6 months is noted.

**Command run:**
- `composer outdated --direct --format=json`

---

### Category 4: Code Quality (0-20)

**Tools:** phpcs (or php-cs-fixer), phpcpd, phploc

Points split proportionally among available tools. All 3 available: phpcs ~7, phpcpd ~7, phploc ~6. Two available: 10 each. One available: full 20.

**phpcs scoring (proportional share):**

| Score | Criteria |
|-------|----------|
| Full | 0 violations |
| 75% | Fewer than 50 violations |
| 50% | Fewer than 200 violations |
| 25% | 200 or more violations |

If phpcs is absent but **php-cs-fixer** is available, `php-cs-fixer fix --dry-run --diff --format=json` provides the violation count.

**phpcpd scoring (proportional share):**

| Score | Criteria |
|-------|----------|
| Full | 0 clones |
| 75% | Fewer than 5 clones |
| 50% | Fewer than 20 clones |
| 25% | 20 or more clones |

**phploc scoring (proportional share):**

| Score | Criteria |
|-------|----------|
| Full | Average class length < 200 LOC AND average method length < 20 LOC |
| 75% | Average class < 300 LOC AND average method < 30 LOC |
| 50% | Average class < 500 LOC AND average method < 40 LOC |
| 25% | Exceeds the above thresholds |

**Commands run:**
- `phpcs --report=json`
- `phpcpd <source-dirs>`
- `phploc <source-dirs>`

---

### Category 5: Test Coverage (0-20)

**Tools:** PHPUnit or Codeception + coverage driver (Xdebug or PCOV)

| Score | Criteria |
|-------|----------|
| 20 | More than 90% line coverage |
| 15 | 70-90% line coverage |
| 10 | 50-70% line coverage |
| 5 | 30-50% line coverage |
| 0 | Less than 30% line coverage |

**Special cases:**
- No coverage driver (neither Xdebug nor PCOV): score 0, note in report
- No test framework at all: score 0, creates a CRITICAL finding

**Commands run:**
- PHPUnit: `phpunit --coverage-text --colors=never`
- Codeception: `codecept run --coverage --coverage-text`

If both are available, the skill uses whichever has more test suites configured.

---

## Part 2: Architectural Audit (9 Dimensions)

Claude-driven analysis using `grep`, `rg`, `find`, and direct code reading. Every finding includes a `file:line` citation. If a pattern is not found, it is not reported.

---

### Dimension 1: Architectural Decay

| Pattern | How Detected |
|---------|-------------|
| God classes (>500 LOC) | Largest files list from orient phase, read for single-class with many responsibilities |
| Service locator abuse | `rg 'Yii::\$app->' ...`, `rg '\bapp\(\)' ...`, `rg 'Container::get\(' ...` outside DI config files |
| Circular namespace dependencies | Analyze `use` statements: if namespace A imports from B and B imports from A |
| Dead code | Unused `use` statements, unreferenced class files |
| Controllers with business logic | Controller methods >30 lines containing DB queries or complex logic |

---

### Dimension 2: Consistency Rot

| Pattern | How Detected |
|---------|-------------|
| Multiple HTTP clients | `rg 'new GuzzleHttp\|new Http\|file_get_contents\(.*http\|curl_init'` |
| Multiple validation approaches | `rg 'Validator::\|->validate\(\|new.*Validator'` |
| Mixed autoloading | Both `psr-4` and `classmap`/`files` in `composer.json` |
| Inconsistent naming | Both camelCase and snake_case method names in same namespace |
| Multiple config formats | `.php`, `.yml`, `.yaml`, `.json`, `.ini`, `.xml` config files for same purpose |

---

### Dimension 3: Type & Contract Debt

| Pattern | How Detected |
|---------|-------------|
| Missing `declare(strict_types=1)` | Count files with vs without the declaration, report ratio |
| Missing return types | Public methods without `: ReturnType` before `{` |
| `mixed` params without validation | `rg 'mixed \$'`, check method body for type checking |
| `@var` without type declarations | Properties using `@var` annotations but lacking PHP type declarations |
| Untyped properties | `(public\|protected\|private) \$` without `:` type hint |

---

### Dimension 4: Test Debt

| Pattern | How Detected |
|---------|-------------|
| Coverage gaps | Critical paths (controllers, services) with low/no coverage from Phase 3 |
| Tests asserting implementation | `rg '->expects\(.*exactly\(\|->method\('` in test files |
| Skipped/commented tests | `rg '@skip\|markTestSkipped\|markTestIncomplete'` |
| Missing integration tests | Test directory contains only unit tests with full mocking |
| Tests without assertions | Test methods lacking `$this->assert*` or `$this->expect*` |

---

### Dimension 5: Dependency Debt

| Pattern | How Detected |
|---------|-------------|
| CVEs | From Phase 3 Security scoring |
| Dev packages in production require | phpunit, phpstan, phpcs, mockery, faker in `require` instead of `require-dev` |
| Unused packages | Required packages whose namespace is never imported in source |
| Unmaintained packages | Abandoned flags from composer |
| PHP version constraint issues | Too loose (`>=7.0`), too tight (`=8.1.5`), or mismatched with runtime |

---

### Dimension 6: Performance

| Pattern | How Detected |
|---------|-------------|
| N+1 queries | ORM/DB calls inside `foreach` loop bodies |
| `file_get_contents()` for HTTP | `rg "file_get_contents\s*\(\s*['\"]https?://"` |
| Missing database indexes | Migration files with foreign keys but no index declarations |
| Synchronous I/O in request path | `sleep()`, `file_get_contents()`, `curl_exec()` in controllers |
| Large arrays without generators | `array_map`/`array_filter`/`array_merge` on potentially large datasets |

---

### Dimension 7: Error Handling

| Pattern | How Detected |
|---------|-------------|
| Empty catch blocks | Catch blocks with empty bodies |
| Catch without logging/re-throw | Catch blocks that neither log nor re-throw |
| Too-broad catches | `catch (\Exception` or `catch (\Throwable` where specific types fit |
| Silent failures | `return null`/`return false` in error paths without logging |
| Error suppression operator | `@file`, `@fopen`, `@mail`, `@unlink`, `@mkdir` |

---

### Dimension 8: Security Hygiene

| Pattern | How Detected |
|---------|-------------|
| Direct superglobal access | `$_GET[`, `$_POST[`, `$_REQUEST[`, `$_SERVER[` outside framework bootstrap |
| SQL string concatenation | String concatenation with SQL keywords (SELECT, INSERT, etc.) |
| Dangerous functions | `eval()`, `extract()`, `exec()`, `system()`, `passthru()`, `shell_exec()` |
| Hardcoded credentials | `password =`, `api_key =`, `secret =`, `token =` with string values |
| Weak password hashing | `md5()` or `sha1()` used for password hashing |
| Missing CSRF protection | Forms/POST endpoints without CSRF token validation |
| Unvalidated redirects | Redirects using `$_GET`/`$_POST` values |

---

### Dimension 9: Documentation Drift

| Pattern | How Detected |
|---------|-------------|
| @param type mismatches | Compare `@param` annotations with actual method signatures |
| Public methods without docblocks | `rg 'public function'` with no preceding `/**` block |
| README vs actual setup | Compare README instructions with real project structure |
| Stale TODO/FIXME comments | `rg 'TODO\|FIXME\|HACK\|XXX'` + `git blame` to check age (>6 months = stale) |
