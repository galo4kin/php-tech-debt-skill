# Tech Debt Audit — {{PROJECT_NAME}}

**Generated:** {{DATE}}
**Auditor:** Claude Code (`php-tech-debt-audit` v1.0)
**Runtime:** {{RUNTIME_MODE}}

---

## Executive Summary

{{EXECUTIVE_SUMMARY}}

---

## Health Score: {{TOTAL_SCORE}}/100 {{SCORE_DELTA}}

| Category | Raw Score | Weight | Status | Tools Used |
|----------|-----------|--------|--------|------------|
| Security | {{SECURITY_RAW}}/20 | {{SECURITY_WEIGHT}}% | {{SECURITY_STATUS}} | {{SECURITY_TOOLS}} |
| Static Analysis | {{STATIC_RAW}}/20 | {{STATIC_WEIGHT}}% | {{STATIC_STATUS}} | {{STATIC_TOOLS}} |
| Dependencies | {{DEPS_RAW}}/20 | {{DEPS_WEIGHT}}% | {{DEPS_STATUS}} | {{DEPS_TOOLS}} |
| Code Quality | {{QUALITY_RAW}}/20 | {{QUALITY_WEIGHT}}% | {{QUALITY_STATUS}} | {{QUALITY_TOOLS}} |
| Test Coverage | {{COVERAGE_RAW}}/20 | {{COVERAGE_WEIGHT}}% | {{COVERAGE_STATUS}} | {{COVERAGE_TOOLS}} |

### Tools Not Available

{{MISSING_TOOLS_LIST}}

---

## Architectural Mental Model

{{ARCHITECTURE_DESCRIPTION}}

---

## Tool Results

### Security

{{SECURITY_DETAILS}}

### Static Analysis

{{STATIC_ANALYSIS_DETAILS}}

### Dependencies

{{DEPENDENCY_DETAILS}}

### Code Quality

{{CODE_QUALITY_DETAILS}}

### Test Coverage

{{COVERAGE_DETAILS}}

---

## Architectural Findings

| ID | Category | File:Line | Severity | Effort | Status | Description | Recommendation |
|----|----------|-----------|----------|--------|--------|-------------|----------------|
{{FINDINGS_TABLE}}

---

## Top 5 Priorities

{{TOP_5_PRIORITIES}}

---

## Quick Wins

{{QUICK_WINS_CHECKLIST}}

---

## Looks Bad But Is Fine

{{LOOKS_BAD_BUT_FINE}}

---

## Open Questions

{{OPEN_QUESTIONS}}

---

## Appendix

### Tools & Versions

{{TOOLS_VERSIONS}}

### Metrics Baseline

```yaml
{{METRICS_YAML}}
```

### Run History

| Date | Score | Δ | Findings | Resolved | New |
|------|-------|---|----------|----------|-----|
{{RUN_HISTORY}}

---

## Previously Resolved

<details>
<summary>{{RESOLVED_COUNT}} resolved findings from previous runs</summary>

{{RESOLVED_FINDINGS}}

</details>
