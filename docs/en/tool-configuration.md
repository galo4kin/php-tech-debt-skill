---
title: Tool Configuration
layout: default
parent: English
nav_order: 4
lang: en
counterpart: /php-tech-debt-skill/ru/tool-configuration/
---

{%- include lang-switcher.html -%}

# Tool Configuration

Every tool is optional. The skill detects what is available and adjusts scoring accordingly. This page explains how each tool is detected, what configuration it reads, and how its settings affect the score.

## Detection Order

For each tool, the skill checks availability in this order:

1. Global binary — `which <tool>`
2. Vendor binary — `test -f vendor/bin/<tool>`
3. Composer require-dev — entry in `composer.json`
4. Config files — tool-specific configuration files in the project root

For Docker environments, all detection commands run via `docker compose exec -T <service>`.

After detection, the skill prints a summary table:

```
Tool Detection Results:
  Runtime         Docker (php-service via docker/docker-compose.yml)
  composer audit  ✓ (composer 2.7.1)
  PHPStan         ✓ (vendor/bin/phpstan, level 6, phpstan.neon)
  semgrep         ✓ (0.185.0)
  phpcs           ✗ (php-cs-fixer detected as alternative)
  PHPUnit         ✓ (vendor/bin/phpunit, phpunit.xml.dist)
  Coverage driver ✓ (pcov)
  phpmd           ✗ → install: composer require --dev phpmd/phpmd
  ...
```

---

## Tool Details

### composer audit

**Purpose:** Detect known CVE vulnerabilities in dependencies.

**Detection:** `which composer` or `test -f composer.phar`. For Docker: `$CMD_PREFIX composer --version`.

**Config files:** None. Uses `composer.lock` automatically.

**Command:** `composer audit --format=json`

**Scoring impact:** Feeds into the Security category (0-20). Critical CVEs = 0 points. Zero CVEs = 20 points.

---

### PHPStan

**Purpose:** Static analysis for type errors, code correctness, and potential bugs.

**Detection:** `which phpstan`, `vendor/bin/phpstan`, or in `composer.json` require-dev.

**Config files:** `phpstan.neon`, `phpstan.neon.dist`, `phpstan.dist.neon`

**Key setting — `level:`**

The `level` field in the PHPStan config directly affects scoring:

| PHPStan Level | Score Impact |
|--------------|-------------|
| Level 6+ with <10 errors | Maximum points |
| Level 4-5 with <50 errors | 75% of points |
| Level 1-3 or >100 errors | 50% of points |
| Level 0 or >500 errors | 25% of points |

If no config file exists and PHPStan runs, level 0 is assumed.

**Command:** `phpstan analyse --error-format=json`

**Scoring impact:** Feeds into Static Analysis category. When phpmd is also available, PHPStan gets 12 of the 20 points. When PHPStan is the only tool, it gets all 20.

---

### semgrep

**Purpose:** Security pattern scanning — injection vulnerabilities, hardcoded secrets, dangerous function usage.

**Detection:** `which semgrep`. Always runs on the host (reads source files directly), even in Docker environments.

**Config files:** None. Uses `--config auto` which applies community rulesets.

**Command:** `semgrep scan --config auto --lang php --json --quiet`

**Install:** `pip install semgrep`

**Scoring impact:** Feeds into the Security category alongside composer audit.

---

### phpcs (PHP_CodeSniffer)

**Purpose:** Coding standards compliance (PSR-12, custom rulesets).

**Detection:** `which phpcs`, `vendor/bin/phpcs`, or in `composer.json` require-dev.

**Config files:** `phpcs.xml`, `.phpcs.xml`, `phpcs.xml.dist`, `.phpcs.xml.dist`

The config file defines which coding standard and rulesets to enforce. The skill uses the project's existing config — it does not impose a default standard.

**Command:** `phpcs --report=json`

**Scoring impact:** Feeds into Code Quality category. Points split proportionally with other available tools (phpcpd, phploc).

---

### php-cs-fixer

**Purpose:** Alternative to phpcs for coding standards enforcement.

**Detection:** `vendor/bin/php-cs-fixer` or in `composer.json` require-dev.

**Config files:** `.php-cs-fixer.dist.php`, `.php-cs-fixer.php`

**When used:** Only when phpcs is not available. The skill runs it in dry-run mode to count violations without modifying files.

**Command:** `php-cs-fixer fix --dry-run --diff --format=json`

**Scoring impact:** Same as phpcs — feeds into Code Quality.

---

### PHPUnit

**Purpose:** Test execution and line coverage measurement.

**Detection:** `which phpunit`, `vendor/bin/phpunit`, or in `composer.json` require-dev.

**Config files:** `phpunit.xml`, `phpunit.xml.dist`

**Requires coverage driver:** Xdebug or PCOV must be installed as a PHP extension. Without a coverage driver, the skill cannot measure coverage and scores 0 for Test Coverage.

**Command:** `phpunit --coverage-text --colors=never`

**Scoring impact:** Feeds into Test Coverage category (0-20).

---

### Codeception

**Purpose:** Alternative test framework. Used when PHPUnit is not the primary framework.

**Detection:** `vendor/bin/codecept` or in `composer.json` require-dev.

**Config files:** `codeception.yml`, `codeception.dist.yml`

**Command:** `codecept run --coverage --coverage-text`

**Scoring impact:** Same as PHPUnit — feeds into Test Coverage.

If both PHPUnit and Codeception are available, the skill uses whichever has more test suites configured.

---

### phpmd (PHP Mess Detector)

**Purpose:** Cyclomatic complexity, design rule violations, unused code detection.

**Detection:** `which phpmd`, `vendor/bin/phpmd`.

**Config files:** `phpmd.xml`, `phpmd.xml.dist`

**Rulesets applied:** `cleancode`, `codesize`, `controversial`, `design`, `naming`, `unusedcode`

**Command:** `phpmd <source-dirs> json cleancode,codesize,controversial,design,naming,unusedcode`

Source directories come from PSR-4 autoload in `composer.json`. Fallback: `src/,app/,lib/`.

**Scoring impact:** Feeds into Static Analysis alongside PHPStan.

---

### phpcpd (Copy/Paste Detector)

**Purpose:** Detect duplicated code blocks across the codebase.

**Detection:** `which phpcpd`, `vendor/bin/phpcpd`.

**Config files:** None.

**Command:** `phpcpd <source-dirs>`

**Scoring impact:** Feeds into Code Quality.

---

### phploc

**Purpose:** Codebase size statistics — lines of code, average class length, average method length.

**Detection:** `which phploc`, `vendor/bin/phploc`.

**Config files:** None.

**Command:** `phploc <source-dirs>`

**Key metrics for scoring:** Average class length (LOC) and average method length (LOC). Smaller = better.

**Scoring impact:** Feeds into Code Quality.

---

### pdepend

**Purpose:** Software quality metrics and dependency analysis.

**Detection:** `which pdepend`, `vendor/bin/pdepend`.

**Config files:** None.

**Note:** Currently detected but not directly used in scoring. Future versions may incorporate pdepend metrics.

---

### Coverage Driver (Xdebug / PCOV)

**Purpose:** Required by PHPUnit/Codeception to measure line coverage.

**Detection:** `php -m | grep -iE 'xdebug|pcov'`

**Recommendation:** PCOV is faster than Xdebug for coverage-only use. Install with `pecl install pcov`.

If neither driver is available, Test Coverage scores 0 with a note in the report.

---

## Which Tools to Install First

For maximum scoring coverage with minimum setup:

1. **PHPStan** — enables Static Analysis scoring. `composer require --dev phpstan/phpstan`
2. **PHPUnit** + **PCOV** — enables Test Coverage scoring. Most PHP projects already have PHPUnit.
3. **semgrep** — enables deeper Security scanning. `pip install semgrep`
4. **phpcs** or **php-cs-fixer** — enables part of Code Quality scoring.
5. **phpcpd** — enables duplicate detection in Code Quality.

Composer audit and `composer outdated` come built into Composer, so Security and Dependencies categories work with no extra installs.
