---
title: FAQ
layout: default
parent: English
nav_order: 6
lang: en
counterpart: /ru/faq/
---

{%- include lang-switcher.html -%}

# FAQ

## "No PHP runtime detected"

The skill could not find PHP on the host (`which php` returned nothing) and could not find a running Docker container with PHP.

**Solutions:**
- Install PHP on your machine
- If using Docker: make sure your container is running (`docker compose up -d`) and that the compose file is in the project root or `docker/` directory
- If PHP is elsewhere: the skill will still run Claude-driven architectural analysis using file reading only — tool-based scoring will be skipped

---

## "Score seems low but the code is fine"

Several possible reasons:

**Missing tools.** If only 2 out of 5 categories have tools, the normalization makes each category count for more. One weak area drags the total down disproportionately. Install more tools to get a balanced picture.

**Strict rubrics.** The scoring thresholds are deliberately opinionated. For example, Static Analysis gives maximum points only at PHPStan level 6+ with fewer than 10 errors. A project at level 5 with 30 errors scores 75% — solid, but not maximum.

**Outdated dependencies.** The Dependencies category checks what percentage of direct dependencies are outdated. Even if the project works fine, many outdated packages lower the score.

Check the "Missing Tools" section in the report for what to install to unlock deeper analysis.

---

## "How do I improve my score fastest?"

1. **Install missing tools.** Each new tool unlocks a scoring category or adds depth to an existing one. Start with PHPStan and PCOV.
2. **Fix quick wins.** The report lists findings that take under 2 hours and have meaningful impact.
3. **Raise PHPStan level.** Going from level 0 to level 4 can double your Static Analysis score.
4. **Update dependencies.** Run `composer update` for patch/minor updates. Each updated package improves the Dependencies score.

---

## "Can I audit just part of the codebase?"

Yes. Pass a directory argument:

```
/php-tech-debt-audit src/Billing/
```

This scopes the orient phase (directory structure, largest files) to that directory. Tool-based scoring still runs on the full project because tools like PHPStan and phpcs use project-level configuration.

---

## "How often should I run the audit?"

**Recommended cadence:**
- Once per sprint during active development
- Before major releases
- After large refactoring efforts
- When onboarding to an unfamiliar codebase

The repeat-run diffing tracks trends over time, so regular runs build a useful history.

---

## Docker: "Container not running"

The skill found a `docker-compose.yml` but the PHP container is not running.

**Fix:** Start the container before running the audit:

```bash
docker compose up -d
```

Or, if the compose file is in a subdirectory:

```bash
docker compose -f docker/docker-compose.yml up -d
```

---

## Docker: "PHP service not found"

The skill could not identify which service in the compose file runs PHP. It looks for services with `php` in the name.

**Fix:** Make sure your PHP service name contains the word "php" (e.g., `php`, `php-fpm`, `app-php`). If your service has a different name, the skill may not detect it automatically.

---

## Large Repositories (>500 PHP files)

For large codebases, the skill automatically:

1. Identifies top-level modules or directories
2. Dispatches parallel subagents — one per module for the architectural audit (Phase 4)
3. Merges results into a single unified report

Tool-based scoring (Phase 3) always runs centrally on the full project. Only the architectural audit is parallelized.

This behavior is automatic — no configuration needed.

---

## "What about the HTML report?"

An interactive HTML report with charts and collapsible sections is planned for a future release. Currently, the skill produces Markdown only (`TECH_DEBT_AUDIT.md`).
