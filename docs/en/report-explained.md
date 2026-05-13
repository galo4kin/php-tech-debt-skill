---
title: Understanding the Report
layout: default
parent: English
nav_order: 5
lang: en
counterpart: /php-tech-debt-skill/ru/report-explained/
---

{%- include lang-switcher.html -%}

# Understanding the Report

After the audit completes, the skill writes `TECH_DEBT_AUDIT.md` to the project root. This page explains every section of that report.

---

## Executive Summary

Two to three sentences covering overall project health, the single biggest concern, and the recommended first action. This is the section to share with stakeholders who will not read the full report.

---

## Health Score Card

A table showing each of the 5 scoring categories:

| Column | Meaning |
|--------|---------|
| Category | Security, Static Analysis, Dependencies, Code Quality, or Test Coverage |
| Raw Score | Points earned out of 20 for this category |
| Weight | Percentage weight after normalization (higher if fewer categories have tools) |
| Status | ✓ (15-20, healthy), ⚠ (8-14, needs attention), or ✗ (0-7, critical) |
| Tools Used | Which tools produced data for this category |

The total score at the top is the normalized value out of 100.

### Missing Tools

Listed below the score card. Each missing tool shows what it would check and how to install it. Installing more tools means deeper analysis on future runs.

---

## Architectural Mental Model

A 1-2 paragraph summary describing what the application does, its architecture (monolith, modular monolith, microservice), the framework and patterns used, and notable structural decisions. This section is written during the orient phase before any analysis begins.

---

## Tool Results

Detailed output for each scoring category:

- **Security:** List of CVEs from composer audit (advisory ID, package, severity, affected version). Semgrep security findings with file:line references.
- **Static Analysis:** PHPStan errors grouped by file. phpmd violations by rule.
- **Dependencies:** Table of outdated packages with current version, latest version, and whether the gap is major/minor/patch. Notes on composer.lock age.
- **Code Quality:** phpcs violation summary. phpcpd clone list. phploc size statistics.
- **Test Coverage:** Coverage percentage. List of critical paths (controllers, services) with low or no coverage.

---

## Architectural Findings

The main findings table with these columns:

| Column | Meaning |
|--------|---------|
| ID | Unique identifier (F001, F002, ...). Stable across repeat runs. |
| Category | Which of the 9 audit dimensions this belongs to |
| File:Line | Exact location in the codebase |
| Severity | `critical`, `high`, `medium`, or `low` |
| Effort | Estimated hours to fix |
| Status | `ACTIVE`, `NEW`, `RESOLVED`, or `SHIFTED` (on repeat runs) |
| Description | What the issue is |
| Recommendation | How to fix it (scoped and concrete, never a rewrite proposal) |

Findings are sorted by severity descending, then by effort ascending (cheapest fixes first within each severity level).

---

## Top 5 Priorities

The 5 most impactful findings, ranked by severity multiplied by effort efficiency (high severity + low effort = high priority). Each includes a brief justification for why it was chosen.

---

## Quick Wins

Findings that can be fixed in under 2 hours and have meaningful impact. These are meant to be tackled first — they produce visible improvement with minimal investment.

---

## Looks Bad But Is Fine

Patterns that appear problematic but are deliberate design choices or acceptable tradeoffs. For example, a god class that is actually a well-structured facade, or `eval()` used in a sandboxed templating context. Each entry explains why it is acceptable.

This section prevents future developers from "fixing" things that are not broken.

---

## Open Questions

Uncertainties that require input from the maintainers. Patterns where intent is unclear — the skill flags them here instead of making assumptions about whether they are problems.

---

## Appendix: Tools & Versions

Table of every tool that ran during the audit, its version, and the exact command used.

---

## Appendix: Metrics Baseline

A YAML block storing raw numeric values from this run:

```yaml
last_run: "2026-05-12"
score: 72
security_raw: 18
static_analysis_raw: 14
dependencies_raw: 12
code_quality_raw: 16
test_coverage_raw: 12
findings_total: 47
findings_critical: 2
findings_high: 8
findings_resolved_cumulative: 12
phpstan_level: 6
phpstan_errors: 23
coverage_percent: 68.5
```

This block is machine-readable and intended for automation, CI integration, or trend tracking scripts.

---

## Appendix: Run History

A table that grows with each audit run:

| Date | Score | Delta | Findings | Resolved | New |
|------|-------|-------|----------|----------|-----|
| 2026-05-12 | 72 | — | 47 (2/8/22/15) | — | 47 |
| 2026-05-26 | 78 | +6 | 39 (1/6/19/13) | 12 | 4 |

---

## Previously Resolved

A collapsed section at the bottom containing findings from previous runs that are no longer detected. Each shows the original finding ID, description, and resolution date. This preserves the audit trail.

---

## Repeat-Run Statuses

When the skill runs on a project that already has a `TECH_DEBT_AUDIT.md`:

| Status | Meaning |
|--------|---------|
| **ACTIVE** | Previously reported finding is still present at the same location |
| **RESOLVED** | Previously reported finding is no longer detected. Moves to "Previously Resolved" section. |
| **SHIFTED** | Same issue exists but the line number changed (code was added/removed above it). Line reference updates, finding ID stays the same. |
| **NEW** | Issue found in the current run that was not present in the previous run. Gets the next available ID. |

The score line shows the delta: `72/100 (+8 since 2026-03-15)`.
