# [4.14] DevSecOps Practicals - Dependabot and OWASP ZAP

## Dependencies
- [Self Studies](./studies.md)
- [Lesson](./lesson.md)
- [Assignment](./assignment.md)

## Lesson Objectives
* **Turn on** Dependabot for a Maven project - alerts, security updates and automatic dependency submission
* **Distinguish** blocking security checks from non-blocking security checks
* **Triage** a Dependabot alert - dismiss it with a reason, and compare it with the `.snyk` ignore from Lesson 4.13
* **Run** an OWASP ZAP baseline (DAST) scan against a deployed application and interpret its alerts

## Lesson Plan

| Duration | What | How or Why |
|----------|------|------------|
| 10 min | Warm up | Recap 4.13: blocking Snyk checks in CircleCI, the accepted finding in `.snyk`; check prerequisites (public repo, APP URL) |
| 15 min | Part 1: Blocking and non-blocking security checks | Blocking vs non-blocking checks; why Dependabot is non-blocking; continuous monitoring of new advisories |
| 55 min | Part 2: Turn on Dependabot | Dependency graph, automatic dependency submission (transitive Maven dependencies), Dependabot alerts and security updates; watch the submission run; read alerts; look at a security-update PR (don't merge); `.github/dependabot.yml` for pinned GitHub Actions; dismiss one alert with a reason and compare with `.snyk` |
| 10 min | Break | - |
| 55 min | Part 3: OWASP ZAP baseline scan | Baseline vs full scan, scan only what you own; `APP_URL` repository variable; `zap-baseline-scan` workflow pinned by SHA; run it, read the report (job summary / `report_html.html`); discussion |
| 15 min | Part 4: Your complete DevSecOps setup | Blocking Snyk checks, Dependabot and ZAP side by side; best practices |
| 15 min | Recap and wrap up | Summary, troubleshooting, Q&A |
| **Total** | | **175 min - allows ~5 min buffer** |
