---
title: English
layout: default
has_children: true
nav_order: 1
lang: en
counterpart: /php-tech-debt-skill/ru/
---

{%- include lang-switcher.html -%}

# PHP Tech Debt Skill

A Claude Code skill that audits PHP projects for technical debt. It combines automated tool-based scoring with deep architectural analysis to produce an actionable report.

## Key Features

**Tool-based health score (0-100)** — runs up to 11 static analysis and quality tools (PHPStan, semgrep, phpcs, PHPUnit, and others) and scores across 5 categories: Security, Static Analysis, Dependencies, Code Quality, and Test Coverage. Each category is worth 20 points. Missing tools are handled through automatic score normalization.

**9-dimension architectural audit** — Claude-driven analysis that searches for concrete patterns: god classes, N+1 queries, empty catch blocks, missing strict_types, hardcoded credentials, and dozens more. Every finding includes a `file:line` citation.

**Repeat-run tracking** — run the audit again and the report shows what was resolved, what is new, and how the score changed. Findings get statuses: ACTIVE, RESOLVED, SHIFTED, or NEW.

**Docker support** — auto-detects containerized PHP environments. No configuration needed.

## Example Output

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

## Next Steps

- [Getting Started](getting-started) — installation and first run
- [Checks Reference](checks-reference) — all scoring categories and audit dimensions
- [Tool Configuration](tool-configuration) — how each tool is detected and configured
- [Understanding the Report](report-explained) — how to read TECH_DEBT_AUDIT.md
- [FAQ](faq) — troubleshooting and common questions

---

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://github.com/galo4kin/php-tech-debt-skill/blob/main/LICENSE)
