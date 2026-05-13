---
name: php-tech-debt-audit
description: >
  Comprehensive PHP technical debt audit. Tool-based scoring (100-point scale)
  + deep architectural analysis across 9 dimensions. Produces TECH_DEBT_AUDIT.md
  with file:line citations and RESOLVED/NEW tracking on repeat runs.
argument-hint: [directory]
disable-model-invocation: true
allowed-tools:
  - Bash(composer*)
  - Bash(php*)
  - Bash(vendor/bin/*)
  - Bash(./vendor/bin/*)
  - Bash(phpstan*)
  - Bash(phpcs*)
  - Bash(phpmd*)
  - Bash(phpcpd*)
  - Bash(phploc*)
  - Bash(pdepend*)
  - Bash(phpunit*)
  - Bash(codecept*)
  - Bash(semgrep*)
  - Bash(docker*)
  - Bash(git*)
  - Bash(find*)
  - Bash(grep*)
  - Bash(rg*)
  - Bash(wc*)
  - Bash(ls*)
  - Bash(cat*)
  - Bash(head*)
  - Bash(which*)
  - Bash(test*)
  - Read
  - Glob
  - Grep
  - Write
---

# PHP Tech Debt Audit

## Operating Principles

- Every concrete finding MUST have a `file:line` citation. No vague claims allowed.
- No generic platitudes. All findings must be grounded in THIS specific codebase.
- No rewrite proposals. Only scoped, concrete recommendations.
- Score only what can be measured with available tools.
- If unsure about intent behind a pattern, add it to the "Open Questions" section instead of marking it as a finding.
- Run tools with JSON output format where available for reliable parsing.

---

## Phase 1: Environment & Tool Detection

### Step 1 — Runtime Detection (Host vs Docker)

Determine whether PHP runs on the host or inside a Docker container.

1. Run `which php` on the host.
2. If PHP is found on the host, set `CMD_PREFIX=""` (all commands run directly).
3. If PHP is NOT found on the host:
   a. Search for docker-compose files in these locations:
      - `./docker-compose.yml`
      - `./docker-compose.yaml`
      - `./docker/docker-compose.yml`
      - `./docker/docker-compose.yaml`
   b. Find the PHP service by grepping the compose file for services with `php` in the name.
   c. Check that the container is running: `docker compose -f <path> ps`
   d. Set `CMD_PREFIX="docker compose -f <path> exec -T <service>"`
      The `-T` flag disables pseudo-TTY allocation, which is required for non-interactive execution.
   e. Verify PHP works: `$CMD_PREFIX php -v`
4. If neither host PHP nor Docker PHP is available:
   - Warn the user: "No PHP runtime detected. Tool-based scoring will be skipped. Performing Claude-driven code analysis only."
   - Skip all of Phase 3 (tool-based scoring).
   - Proceed with Phase 2 (Orient) and Phase 4 (Architectural Audit) using file reading only.

### Step 2 — Tool Detection

For each tool below, check availability in this order: global binary (`which <tool>`), vendor binary (`test -f vendor/bin/<tool>` or via CMD_PREFIX), composer.json require-dev section, and config files.

For Docker runtime: run all detection commands via `CMD_PREFIX`. Example:
`docker compose -f docker/docker-compose.yml exec -T php-service php vendor/bin/phpstan --version`

**Tools to detect:**

1. **composer audit** — Check: `which composer` or `test -f composer.phar`. For Docker: `$CMD_PREFIX composer --version`.
2. **PHPStan** — Check binary: `which phpstan`, `test -f vendor/bin/phpstan`, or in composer.json require-dev. Check config files: `phpstan.neon`, `phpstan.neon.dist`, `phpstan.dist.neon`. If config found, parse it for the `level:` field to record the configured level.
3. **semgrep** — Check: `which semgrep`. This runs on the host regardless of Docker (it reads source files directly).
4. **phpcs** — Check binary: `which phpcs`, `test -f vendor/bin/phpcs`, or in composer.json require-dev. Check config files: `phpcs.xml`, `.phpcs.xml`, `phpcs.xml.dist`, `.phpcs.xml.dist`.
5. **php-cs-fixer** — Check binary: `test -f vendor/bin/php-cs-fixer` or in composer.json require-dev. Check config files: `.php-cs-fixer.dist.php`, `.php-cs-fixer.php`. This serves as an alternative to phpcs for Code Quality scoring.
6. **PHPUnit** — Check binary: `which phpunit`, `test -f vendor/bin/phpunit`, or in composer.json require-dev. Check config files: `phpunit.xml`, `phpunit.xml.dist`.
7. **Codeception** — Check binary: `test -f vendor/bin/codecept` or in composer.json require-dev. Check config files: `codeception.yml`, `codeception.dist.yml`.
8. **Coverage driver** — Run: `$CMD_PREFIX php -m | grep -iE 'xdebug|pcov'`. Record which driver is available (Xdebug, PCOV, or none).
9. **phpmd** — Check binary: `which phpmd`, `test -f vendor/bin/phpmd`. Check config files: `phpmd.xml`, `phpmd.xml.dist`.
10. **phpcpd** — Check binary: `which phpcpd`, `test -f vendor/bin/phpcpd`.
11. **phploc** — Check binary: `which phploc`, `test -f vendor/bin/phploc`.
12. **pdepend** — Check binary: `which pdepend`, `test -f vendor/bin/pdepend`.

### Step 2 Output — Console Summary Table

After detection, print a summary table to the console:

```
Tool Detection Results:
  Runtime         <Host (php X.Y.Z) | Docker (<service> via <compose-path>) | Not found>
  composer audit  <✓ (composer X.Y.Z) | ✗ → install: https://getcomposer.org>
  PHPStan         <✓ (vendor/bin/phpstan, level N, <config-file>) | ✗ → install: composer require --dev phpstan/phpstan>
  semgrep         <✓ (X.Y.Z) | ✗ → install: pip install semgrep>
  phpcs           <✓ (vendor/bin/phpcs, <config-file>) | ✗ (php-cs-fixer detected as alternative) | ✗ → install: composer require --dev squizlabs/php_codesniffer>
  php-cs-fixer    <✓ (vendor/bin/php-cs-fixer, <config-file>) | ✗>
  PHPUnit         <✓ (vendor/bin/phpunit, <config-file>) | ✗ → install: composer require --dev phpunit/phpunit>
  Codeception     <✓ (vendor/bin/codecept, <config-file>) | ✗>
  Coverage driver <✓ (pcov) | ✓ (xdebug) | ✗ → install: pecl install pcov>
  phpmd           <✓ (vendor/bin/phpmd, <config-file>) | ✗ → install: composer require --dev phpmd/phpmd>
  phpcpd          <✓ (vendor/bin/phpcpd) | ✗ → install: composer require --dev sebastian/phpcpd>
  phploc          <✓ (vendor/bin/phploc) | ✗ → install: composer require --dev phploc/phploc>
  pdepend         <✓ (vendor/bin/pdepend) | ✗ → install: composer require --dev pdepend/pdepend>
```

Use ✓ for detected tools, ✗ for missing tools. Include version numbers where available. For missing tools, show the install command.

---

## Phase 2: Orient

Mandatory understanding before any analysis. Build a mental model of the codebase first.

If `$ARGUMENTS` is provided, scope all orient steps to that directory instead of the project root.

### Step 1 — Project Identity

Read `composer.json` and extract:
- Project name (`name` field)
- PHP version constraint (`require.php`)
- Framework (detect from require: `laravel/framework`, `symfony/framework-bundle`, `yiisoft/yii2`, etc.)
- Key dependencies (the top 5-10 most significant packages)

### Step 2 — Documentation

Read `README.md` if it exists. Scan `docs/` directory if it exists. Note:
- Stated architecture or design patterns
- Setup instructions
- Any documented conventions or decisions

### Step 3 — Directory Structure

Run: `find . -type d -maxdepth 3 -not -path './vendor/*' -not -path './node_modules/*' -not -path './.git/*'`

Understand the top-level layout: where is application code, tests, config, migrations, etc.

### Step 4 — Git Churn Analysis

Run these commands:
- Recent history: `git log --oneline -200`
- Top 20 most-changed files in the last 6 months:
  ```
  git log --format='%H' --since='6 months ago' | xargs -I{} git diff-tree --no-commit-id --name-only -r {} | sort | uniq -c | sort -rn | head -20
  ```

### Step 5 — Largest Files

Find the top 20 largest PHP files (excluding vendor):
```
find . -name '*.php' -not -path './vendor/*' -not -path './node_modules/*' | xargs wc -l | sort -rn | head -20
```

### Step 6 — Priority Targets

Cross-reference Step 4 (most churned) with Step 5 (largest files). Files that are BOTH large AND frequently changed are the highest-priority debt targets. List them explicitly.

### Step 7 — Mental Model

Write a 1-2 paragraph summary describing:
- What the application does
- Its architecture (monolith, modular monolith, microservice, etc.)
- The framework and key patterns used
- Notable structural decisions

This mental model will be included in the final report.

---

## Phase 3: Tool-Based Scoring

5 categories, each worth 0-20 points, for a total of 100.

### Normalization

Not all projects have all tools. Use this normalization formula:

```
available_categories = count of categories where at least 1 tool is available
weight_per_category = 100 / available_categories
weighted_score = (raw_score / 20) * weight_per_category
final_score = sum of all weighted_scores, rounded to nearest integer
```

The report shows BOTH the raw score per category (X/20) AND the normalized total (Y/100).

---

### Category 1: Security (0-20)

**Sources:** `composer audit`, `semgrep`

**Commands:**
- `$CMD_PREFIX composer audit --format=json`
- `semgrep scan --config auto --lang php --json --quiet` (runs on host, reads files directly)

**Scoring rubric:**
- **20 points:** Zero CVEs from composer audit AND zero security findings from semgrep
- **15 points:** Only low-severity CVEs, no critical/high semgrep security findings
- **10 points:** Medium-severity CVEs present OR moderate semgrep findings
- **5 points:** High-severity CVEs present
- **0 points:** Critical CVEs present OR critical semgrep findings

If only one tool is available, score based on that tool alone (full 20 points still possible).

Record all findings with advisory IDs, package names, severity, and affected versions for the report.

---

### Category 2: Static Analysis (0-20)

**Sources:** `PHPStan`, `phpmd`

**Commands:**
- `$CMD_PREFIX phpstan analyse --error-format=json` (uses project config automatically)
- `$CMD_PREFIX phpmd <source-dirs> json cleancode,codesize,controversial,design,naming,unusedcode`

For `<source-dirs>`: extract directories from PSR-4 autoload mapping in `composer.json`. Fallback to `src/,app/,lib/` if PSR-4 not configured.

**Scoring when BOTH PHPStan and phpmd are available (12 + 8 = 20):**

PHPStan portion (0-12):
- **12 points:** Level 6+, fewer than 10 errors
- **9 points:** Level 4-5, fewer than 50 errors
- **6 points:** Level 1-3 OR more than 100 errors
- **3 points:** Level 0 OR more than 500 errors

phpmd portion (0-8):
- **8 points:** 0 violations
- **6 points:** Fewer than 20 violations
- **4 points:** Fewer than 100 violations
- **2 points:** 100 or more violations

**Scoring when only ONE tool is available (0-20):**

PHPStan only:
- **20 points:** Level 6+, fewer than 10 errors
- **15 points:** Level 4-5, fewer than 50 errors
- **10 points:** Level 1-3 OR more than 100 errors
- **5 points:** Level 0 OR more than 500 errors

phpmd only:
- **20 points:** 0 violations
- **15 points:** Fewer than 20 violations
- **10 points:** Fewer than 100 violations
- **5 points:** 100 or more violations

PHPStan level detection: parse `phpstan.neon`, `phpstan.neon.dist`, or `phpstan.dist.neon` for the `level:` field. If no config file exists and PHPStan runs, assume level 0.

---

### Category 3: Dependencies (0-20)

**Source:** `$CMD_PREFIX composer outdated --direct --format=json`

**Scoring rubric based on percentage of outdated direct dependencies + major version gaps:**
- **20 points:** Less than 10% outdated, no major version gaps
- **15 points:** Less than 25% outdated OR 1-2 major version gaps
- **10 points:** Less than 50% outdated OR 3+ major version gaps
- **5 points:** More than 50% outdated
- **0 points:** More than 75% outdated OR abandoned packages detected

**Additional check:** `composer.lock` age via `git log -1 --format=%ci composer.lock`. Note if it has not been updated in more than 6 months.

Record all outdated packages with current version, latest version, and whether the gap is major/minor/patch.

---

### Category 4: Code Quality (0-20)

**Sources:** `phpcs`, `phpcpd`, `phploc`

Points are split proportionally among available tools. If all 3 available: phpcs gets ~7, phpcpd gets ~7, phploc gets ~6. If 2 available: 10 each. If 1 available: full 20.

**phpcs scoring (proportional share):**
- **Full points:** 0 violations
- **75%:** Fewer than 50 violations
- **50%:** Fewer than 200 violations
- **25%:** 200 or more violations

Command: `$CMD_PREFIX phpcs --report=json` (uses project config)

**If phpcs is not available but php-cs-fixer is:** Use `$CMD_PREFIX php-cs-fixer fix --dry-run --diff --format=json` to get a violation count. Apply the same scoring rubric as phpcs.

**phpcpd scoring (proportional share):**
- **Full points:** 0 clones detected
- **75%:** Fewer than 5 clones
- **50%:** Fewer than 20 clones
- **25%:** 20 or more clones

Command: `$CMD_PREFIX phpcpd <source-dirs>`

**phploc scoring (proportional share):**
- **Full points:** Average class length under 200 LOC AND average method length under 20 LOC
- **75%:** Average class under 300 LOC AND average method under 30 LOC
- **50%:** Average class under 500 LOC AND average method under 40 LOC
- **25%:** Exceeds the above thresholds

Command: `$CMD_PREFIX phploc <source-dirs>`

---

### Category 5: Test Coverage (0-20)

**Sources:** `PHPUnit` or `Codeception` + coverage driver (Xdebug or PCOV)

**Commands:**
- PHPUnit: `$CMD_PREFIX phpunit --coverage-text --colors=never`
- Codeception: `$CMD_PREFIX codecept run --coverage --coverage-text`

If both PHPUnit and Codeception are available, use whichever the project primarily uses (check which config file has more test suites defined).

**Scoring rubric:**
- **20 points:** More than 90% line coverage
- **15 points:** 70-90% line coverage
- **10 points:** 50-70% line coverage
- **5 points:** 30-50% line coverage
- **0 points:** Less than 30% line coverage

**Special cases:**
- No coverage driver available (neither Xdebug nor PCOV): Score 0, add note "Install Xdebug or PCOV for coverage analysis."
- No test framework at all (neither PHPUnit nor Codeception): Score 0, create a CRITICAL finding in the report: "No test framework detected. The project has zero automated tests."

Parse the coverage percentage from the `--coverage-text` output. Look for the "Lines:" summary line.

---

## Phase 4: Architectural Audit (9 Dimensions)

Claude-driven analysis using `grep`, `rg`, `find`, and direct code reading. Every finding MUST include a `file:line` citation.

For each dimension, use targeted searches to find concrete instances. Do not guess or speculate. If a pattern is not found, do not report it.

---

### Dimension 1: Architectural Decay

Search for these specific patterns:

**God classes (>500 LOC):**
Use the largest files list from Phase 2. Read each file over 500 LOC and check if it contains a single class with too many responsibilities. Record `file:line` where the class is declared.

**Service locator abuse:**
```
rg -n 'Yii::\$app->' --type php -g '!vendor/*'
rg -n '\bapp\(\)' --type php -g '!vendor/*'
rg -n 'Container::get\(' --type php -g '!vendor/*'
```
Exclude files that are DI configuration files (e.g., `config/`, `bootstrap/`, `container.php`).

**Circular namespace dependencies:**
Analyze `use` statements across namespaces. If namespace A imports from namespace B and namespace B imports from namespace A, flag it.

**Dead code:**
- Unused `use` statements: `rg -n '^use ' --type php -g '!vendor/*'` — then check if the imported class is actually referenced in the file.
- Unreferenced classes: check if any class files are never imported or instantiated anywhere else.

**Controllers with business logic:**
Read controller files. If a controller method exceeds 30 lines or contains database queries, business rules, or complex logic (not just request handling and response), flag it.

---

### Dimension 2: Consistency Rot

**Multiple patterns for the same task:**
```
rg -n 'new GuzzleHttp|new Http|file_get_contents\(.*http|curl_init' --type php -g '!vendor/*'
rg -n 'Validator::|->validate\(|new.*Validator' --type php -g '!vendor/*'
```
If multiple HTTP clients or validation approaches are used, flag with file:line for each variant.

**Mixed autoloading:**
Check `composer.json` for both `psr-4` and `classmap` or `files` autoload entries. If PSR-4 and legacy autoloading coexist, flag it.

**Inconsistent naming:**
```
rg -n 'function [a-z]+_[a-z]+' --type php -g '!vendor/*' | head -20
rg -n 'function [a-z]+[A-Z]' --type php -g '!vendor/*' | head -20
```
If both camelCase and snake_case method names exist in the same namespace or layer, flag examples.

**Multiple config formats:**
Check for mixed config formats: `.php`, `.yml`, `.yaml`, `.json`, `.ini`, `.xml` config files serving the same purpose.

---

### Dimension 3: Type & Contract Debt

**Missing strict_types:**
```
find . -name '*.php' -not -path './vendor/*' -not -path './node_modules/*' -exec head -5 {} + | grep -L 'strict_types'
```
Or more precisely: count files with and without `declare(strict_types=1)`:
```
rg -l 'declare\(strict_types=1\)' --type php -g '!vendor/*' | wc -l
find . -name '*.php' -not -path './vendor/*' -not -path './node_modules/*' | wc -l
```
Report the ratio: "X of Y PHP files have strict_types declared."

**Missing return types on public methods:**
```
rg -n 'public function [a-zA-Z]+\([^)]*\)\s*{' --type php -g '!vendor/*' | head -30
```
Look for public methods without `: ReturnType` before the opening brace. Flag examples.

**Mixed params without validation:**
```
rg -n 'mixed \$' --type php -g '!vendor/*'
```
Check if `mixed` typed parameters have validation or type checking inside the method body.

**@var without type declarations:**
```
rg -n '@var\s+' --type php -g '!vendor/*' | head -20
```
Check if properties using `@var` annotations lack actual PHP type declarations (PHP 7.4+).

**Untyped properties:**
```
rg -n '(public|protected|private)\s+\$' --type php -g '!vendor/*' | grep -v ':' | head -20
```
Flag properties declared without type hints (for PHP 7.4+ projects).

---

### Dimension 4: Test Debt

**Coverage gaps:**
If coverage data is available from Phase 3, identify critical paths (controllers, services, repositories) with low or no coverage.

**Tests asserting implementation not behavior:**
Look for tests that mock too many internals or assert on method call counts rather than outcomes:
```
rg -n '->expects\(.*exactly\(|->method\(' --type php -g '!vendor/*' -g 'test*' -g 'Test*'
```

**Skipped or commented tests:**
```
rg -n '@skip|$this->markTestSkipped|markTestIncomplete' --type php -g '!vendor/*'
rg -n '//.*function test|/\*.*function test' --type php -g '!vendor/*'
```

**Missing integration tests:**
Check if the test directory contains only unit tests (mocking everything) but no integration or feature tests.

**Tests without assertions:**
```
rg -n 'function test' --type php -g '!vendor/*' -g '*Test*'
```
Read test methods and check if any lack `$this->assert*`, `$this->expect*`, or PHPUnit assertion calls.

---

### Dimension 5: Dependency Debt

**CVEs:** Already captured in Phase 3 Security scoring. Reference those findings here.

**Packages in require that should be in require-dev:**
Check `composer.json` `require` section for packages that are typically dev-only:
- `phpunit/phpunit`, `phpstan/phpstan`, `squizlabs/php_codesniffer`, `friendsofphp/php-cs-fixer`
- `mockery/mockery`, `fakerphp/faker`, `codeception/*`
- `symfony/debug`, `barryvdh/laravel-debugbar`

**Unused packages:**
For each package in `require`, check if its namespace is actually imported anywhere in the source code:
```
rg -l '<namespace-prefix>' --type php -g '!vendor/*'
```
If no source file imports from a required package, flag it.

**Unmaintained packages:**
Note any packages flagged as abandoned by composer (shown during `composer update` or in `composer.lock` metadata).

**PHP version constraint:**
Check the `require.php` constraint in `composer.json`. Flag if:
- It is too loose (e.g., `>=7.0`) allowing very old PHP versions
- It is too tight (e.g., `=8.1.5`) preventing minor updates
- It does not match the actual runtime PHP version

---

### Dimension 6: Performance

**N+1 queries (queries in loops):**
```
rg -n 'foreach.*{' --type php -g '!vendor/*' -A 10 | grep -E '->find\(|->query\(|::find\(|->select\(|->where\('
```
Also search for ORM calls inside loop bodies.

**file_get_contents for HTTP:**
```
rg -n "file_get_contents\s*\(\s*['\"]https?://" --type php -g '!vendor/*'
```

**Missing database indexes:**
Search migration files for column definitions that are likely queried but lack index declarations:
```
rg -n 'foreign\|references\|->index\|->unique' --type php -g '!vendor/*' -g '*migration*'
```
Look for foreign key columns without corresponding index.

**Synchronous I/O in request path:**
Look for `sleep()`, `file_get_contents()`, `curl_exec()`, `fopen()` in controller or request-handling code.

**Large arrays without generators:**
```
rg -n 'array_map\|array_filter\|array_merge' --type php -g '!vendor/*' | head -20
```
Check if these operate on potentially large datasets where generators (`yield`) would be more memory-efficient.

---

### Dimension 7: Error Handling

**Empty catch blocks:**
```
rg -n 'catch\s*\([^)]+\)\s*\{' --type php -g '!vendor/*' -A 2 | grep -B1 '^\s*}'
```
Or use a multi-line search to find catch blocks with empty bodies.

**Catch without logging or re-throw:**
```
rg -n 'catch\s*\(' --type php -g '!vendor/*' -A 5
```
Read catch blocks and flag those that neither log the exception nor re-throw it.

**Too-broad catches:**
```
rg -n 'catch\s*\(\\?\\?Exception\s' --type php -g '!vendor/*'
rg -n 'catch\s*\(\\?\\?Throwable\s' --type php -g '!vendor/*'
```
Flag catch blocks that catch `\Exception` or `\Throwable` when a more specific exception type would be appropriate.

**Silent failures:**
```
rg -n 'return null;|return false;' --type php -g '!vendor/*'
```
Check context: if these appear in catch blocks or error paths without any logging, flag them.

**Error suppression operator:**
```
rg -n '@\$|@file|@fopen|@mail|@unlink|@mkdir' --type php -g '!vendor/*'
```
Flag uses of the `@` error suppression operator.

---

### Dimension 8: Security Hygiene

**Direct superglobal access:**
```
rg -n '\$_GET\[|\$_POST\[|\$_REQUEST\[|\$_SERVER\[' --type php -g '!vendor/*'
```
Exclude framework bootstrap files and index.php. Flag direct access outside of request abstraction layers.

**String concatenation in SQL:**
```
rg -n '"\s*\.\s*\$.*(?:SELECT|INSERT|UPDATE|DELETE|WHERE|FROM)' --type php -g '!vendor/*' -i
rg -n "'\s*\.\s*\$.*(?:SELECT|INSERT|UPDATE|DELETE|WHERE|FROM)" --type php -g '!vendor/*' -i
```
Flag any SQL query built with string concatenation instead of parameterized queries.

**Dangerous functions:**
```
rg -n '\beval\s*\(|\bextract\s*\(|\bexec\s*\(|\bsystem\s*\(|\bpassthru\s*\(|\bshell_exec\s*\(' --type php -g '!vendor/*'
```

**Hardcoded credentials:**
```
rg -n "password\s*=\s*['\"]|api_key\s*=\s*['\"]|secret\s*=\s*['\"]|token\s*=\s*['\"]\w+" --type php -g '!vendor/*' -i
```
Exclude config files that reference environment variables (e.g., `env('SECRET')`).

**Weak hashing for passwords:**
```
rg -n 'md5\s*\(|sha1\s*\(' --type php -g '!vendor/*'
```
Check context: flag only if used for password hashing, not for checksums or cache keys.

**Missing CSRF protection:**
Check forms and POST endpoints for CSRF token validation. Framework-specific:
- Laravel: look for `@csrf` or `csrf_field()` in blade templates, `VerifyCsrfToken` middleware
- Symfony: look for `csrf_token()` in twig templates
- Yii: look for `Html::hiddenInput(Yii::$app->request->csrfParam)`

**Unvalidated redirects:**
```
rg -n 'redirect\(.*\$_GET|redirect\(.*\$_POST|redirect\(.*\$_REQUEST|header\(.*Location.*\$' --type php -g '!vendor/*'
```

---

### Dimension 9: Documentation Drift

**@param types not matching signatures:**
Read a sample of files with docblocks. Compare `@param` type annotations with actual method parameter types. Flag mismatches with file:line.

**Public methods without docblocks:**
```
rg -n 'public function' --type php -g '!vendor/*' -B 3
```
Check if the lines preceding `public function` contain a docblock (`/**`). Count methods with and without docblocks. Flag specific examples.

**README vs actual setup:**
Compare the setup instructions in README.md with the actual project structure. Flag discrepancies (e.g., README says `php artisan serve` but the project uses Docker; README references files that do not exist).

**Stale TODO/FIXME comments:**
```
rg -n 'TODO|FIXME|HACK|XXX' --type php -g '!vendor/*'
```
For each TODO/FIXME found, run `git blame -L <line>,<line> <file>` to check the date. Flag any older than 6 months.

---

## Phase 5: Deliverable

### Write TECH_DEBT_AUDIT.md

Use the `report-template.md` (sibling file in this skill directory) as the template. Fill ALL `{{PLACEHOLDER}}` variables with data collected during Phases 1-4.

The report MUST contain these sections in order:

1. **Executive Summary** — 2-3 sentences covering overall health, the single biggest concern, and the recommended first action.
2. **Health Score Card** — Table with columns: Category, Raw Score, Weight, Weighted Score, Status Icon, Tools Used.
3. **Missing Tools** — List of tools not detected, with install commands, and what deeper analysis they would enable.
4. **Architectural Mental Model** — The 1-2 paragraph summary from Phase 2 Step 7.
5. **Tool Results** — Detailed output per scoring category:
   - Security: CVE list, semgrep findings
   - Static Analysis: PHPStan errors by file, phpmd violations
   - Dependencies: outdated packages table
   - Code Quality: phpcs/phpcpd/phploc summaries
   - Test Coverage: coverage percentage, uncovered critical paths
6. **Architectural Findings** — Table with columns: ID (F001, F002...), Category, File:Line, Severity (critical/high/medium/low), Effort (hours estimate), Description, Recommendation. Sort by severity descending, then by effort ascending.
7. **Top 5 Priorities** — The 5 most impactful findings ranked by (severity x effort efficiency). Each with a brief justification.
8. **Quick Wins** — Findings that can be fixed in under 2 hours and have meaningful impact. These are the "do these first" items.
9. **Looks Bad But Is Fine** — Patterns that appear problematic but are deliberate design choices or acceptable tradeoffs. Explain why they are acceptable.
10. **Open Questions** — Uncertainties that require maintainer input. Things where intent is unclear.
11. **Appendix: Tools & Versions** — Table of every tool that ran, its version, and the command used.
12. **Appendix: Metrics Baseline** — YAML block for repeat-run diffing:
    ```yaml
    last_run: "YYYY-MM-DD"
    score: <normalized-score>
    security_raw: <0-20>
    static_analysis_raw: <0-20>
    dependencies_raw: <0-20>
    code_quality_raw: <0-20>
    test_coverage_raw: <0-20>
    findings_total: <count>
    findings_critical: <count>
    findings_high: <count>
    findings_resolved_cumulative: <count>
    phpstan_level: <0-9>
    phpstan_errors: <count>
    coverage_percent: <float>
    ```
13. **Appendix: Run History** — Table with columns: Date, Score, Delta, Findings (total/critical/high/medium/low), Resolved.

### Console Summary

After writing the report, print this console summary:

```
PHP Tech Debt Audit Complete
Health Score: XX/100
  Security:        XX/20 ✓/⚠/✗
  Static Analysis: XX/20 ✓/⚠/✗
  Dependencies:    XX/20 ✓/⚠/✗
  Code Quality:    XX/20 ✓/⚠/✗
  Test Coverage:   XX/20 ✓/⚠/✗

Findings: N total (X critical, Y high, Z medium, W low)
Quick Wins: N items
Report: TECH_DEBT_AUDIT.md
```

**Status icon rules:**
- ✓ = score 15-20 (healthy)
- ⚠ = score 8-14 (needs attention)
- ✗ = score 0-7 (critical)

---

## Repeat-Run Diffing

When `TECH_DEBT_AUDIT.md` already exists in the project root, perform a diff-aware audit:

### Step 1 — Parse Existing Report

Read the existing `TECH_DEBT_AUDIT.md` and extract:
- All findings by their ID (F001, F002, F003...)
- The file:line and category of each finding
- The Metrics Baseline YAML block
- The previous score and run date

### Step 2 — Run Full Audit

Execute all phases (1-4) as normal, collecting fresh data.

### Step 3 — Diff Findings

Compare old findings against new results:

- **ACTIVE:** Same file:line + same category is still broken in the new run. Keep the original finding ID. Set status to `ACTIVE`.
- **RESOLVED:** A finding from the previous run is no longer detected (the issue at that file:line is fixed). Keep the original ID. Set status to `RESOLVED`. Add resolution date (today).
- **SHIFTED:** The same issue exists but the line number changed (code was added/removed above it). Update the line number. Keep the original ID. Set status to `ACTIVE`.
- **NEW:** An issue found in the new run that was not present in the previous run. Assign the next available ID (continuing from the highest existing ID). Set status to `NEW`.

### Step 4 — Update Report

- Move `RESOLVED` findings to a collapsed "Previously Resolved" section at the bottom of the Architectural Findings table.
- Show score delta: `XX/100 (+N since YYYY-MM-DD)` or `XX/100 (-N since YYYY-MM-DD)`.
- Append a new row to the Run History table.
- Update the Metrics Baseline YAML with current values.
- Increment `findings_resolved_cumulative` by the number of newly resolved findings.

---

## Large Repo Handling

Before starting Phase 4, check repository size:

```
find . -name '*.php' -not -path './vendor/*' -not -path './node_modules/*' | wc -l
```

If the project has more than 500 PHP files OR more than 50,000 lines of PHP code:

1. **Identify top-level modules:** Determine the major directories or bounded contexts (e.g., `src/Billing/`, `src/Auth/`, `src/Orders/`).
2. **Dispatch parallel subagents:** Send one subagent per module to perform Phase 4 (Architectural Audit) only. Each subagent receives:
   - The module directory path
   - The list of 9 dimensions to audit
   - Instructions to return findings with `file:line` citations
3. **Merge results:** The main agent collects all subagent findings, deduplicates (same file:line should not appear twice), assigns finding IDs (F001...) in a unified sequence, and writes the single unified `TECH_DEBT_AUDIT.md`.

Tool-based scoring (Phase 3) always runs centralized on the full project — it is not split by module.

---

## Phase 2 (Future): HTML Report

<!-- NOT YET IMPLEMENTED — Planned for future release.

Self-contained `tech-debt-report.html` — a single file with inline CSS and JavaScript:
- Radar chart visualizing the 5 scoring categories
- Progress bars for each category score
- Collapsible sections for each architectural dimension
- Severity color coding (critical=red, high=orange, medium=yellow, low=blue)
- Finding details with file:line links
- Score trend chart from Run History data
- No external dependencies — all SVG charts rendered inline
- Generated from the same data used for the Markdown report
- Should be added to .gitignore

When implemented, add a flag or automatic generation step at the end of Phase 5
that produces the HTML report alongside TECH_DEBT_AUDIT.md.
-->
