---
title: Getting Started
layout: default
parent: English
nav_order: 2
lang: en
counterpart: /php-tech-debt-skill/ru/getting-started/
---

{%- include lang-switcher.html -%}

# Getting Started

## Prerequisites

- [Claude Code](https://claude.ai/code) installed and working
- A PHP project to audit

No PHP tools are required on your machine. The skill detects what is available and adjusts accordingly. If no tools are found, it still performs Claude-driven architectural analysis using file reading.

## Installation

All three methods place the skill where Claude Code can discover it. No other configuration is needed.

### Method 1 — Per-project (recommended)

```bash
mkdir -p .claude/skills
git clone https://github.com/galo4kin/php-tech-debt-skill.git .claude/skills/php-tech-debt-audit
```

This installs the skill for one project only. Add `.claude/skills/php-tech-debt-audit/` to your `.gitignore` if you do not want it committed.

### Method 2 — Global (all projects)

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/galo4kin/php-tech-debt-skill.git ~/.claude/skills/php-tech-debt-audit
```

This makes the skill available in every project you open with Claude Code.

### Method 3 — curl (single file)

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

## What Happens When You Run It

The skill executes in 5 phases:

**Phase 1 — Environment Detection.** Checks whether PHP runs on the host or inside a Docker container. Scans for 12 analysis tools (PHPStan, semgrep, phpcs, PHPUnit, etc.) and reports what is available.

**Phase 2 — Orient.** Reads `composer.json`, `README.md`, directory structure, git history, and largest files. Builds a mental model of the codebase before analyzing anything.

**Phase 3 — Tool-Based Scoring.** Runs every detected tool and scores across 5 categories (Security, Static Analysis, Dependencies, Code Quality, Test Coverage). Each category is worth 0-20 points. Total normalizes to 100.

**Phase 4 — Architectural Audit.** Claude-driven analysis across 9 dimensions using targeted searches (`grep`, `rg`, `find`). Every finding gets a `file:line` citation, severity level, and effort estimate.

**Phase 5 — Report.** Writes `TECH_DEBT_AUDIT.md` to the project root with all findings, scores, priorities, and quick wins. Prints a console summary.

## Docker Support

If PHP is not available on the host, the skill automatically:

1. Searches for `docker-compose.yml` in the project root and `docker/` directory
2. Identifies the PHP service by name
3. Verifies the container is running
4. Runs all PHP and Composer commands through `docker compose exec -T <service>`

No configuration needed. Tools that read source files directly (like semgrep) continue to run on the host.

If neither host PHP nor Docker PHP is found, tool-based scoring is skipped and only Claude-driven analysis runs.
