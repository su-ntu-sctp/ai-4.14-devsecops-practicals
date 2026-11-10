# Assignment (Optional)

## Brief

Add non-blocking security checks to your repository on GitHub, and practise managing a finding.

1. **Turn On Dependabot**
   - Use your devops-demo repository (public) with the Lesson 4.13 CircleCI pipeline (a failing snyk_scan is expected)
   - In **Settings → Advanced Security**, turn on the dependency graph, **automatic dependency submission**, **Dependabot alerts** and **Dependabot security updates**
   - Check that the dependency submission run finishes in the **Actions** tab, and that **Insights → Dependency graph** lists transitive libraries
   - Add `.github/dependabot.yml` with a weekly `github-actions` update, commit and push
   - Take screenshots showing:
     - The Dependabot alerts in the **Security** tab
     - The dependency graph with at least one transitive library
     - A Dependabot pull request, if one was opened
   - Write a brief explanation (3-4 sentences):
     - Why Dependabot is non-blocking, while `snyk_scan` in CircleCI is a blocking check
     - Why automatic dependency submission matters for a Spring Boot `pom.xml`

2. **Triage an Alert**
   - Pick one Dependabot alert - ideally the issue you accepted in `.snyk` in Lesson 4.13
   - Dismiss it with a reason and a comment that says why, and when it will be reviewed again
   - Write a brief report (4-5 sentences):
     - The vulnerability ID, severity and patched version (if any)
     - Why you accepted it temporarily, and who fixes it
     - How a Dependabot dismissal differs from an ignore in `.snyk`
     - When ignoring a finding is acceptable, and when it isn't

3. **Run a DAST Scan (Stretch)**
   - Add `.github/workflows/zap-baseline.yml` using `zaproxy/action-baseline` (pinned by commit SHA) against your live app URL (Render or Railway), stored as the repository variable `APP_URL`
   - Run it manually and open the `report_html.html` artifact
   - List the three highest-risk alerts and the solution ZAP suggests for each

## Submission (Optional)

- Submit the URL of the GitHub Repository that contains your work to NTU black board.
- Should you reference the work of your classmate(s) or online resources, give them credit by adding either the name of your classmate or URL.

## References
- Dependabot alerts: https://docs.github.com/en/code-security/dependabot/dependabot-alerts/about-dependabot-alerts
- Automatic dependency submission: https://docs.github.com/en/code-security/how-tos/secure-your-supply-chain/secure-your-dependencies/configuring-automatic-dependency-submission-for-your-repository
- OWASP ZAP baseline scan: https://www.zaproxy.org/docs/docker/baseline-scan/
- OWASP: https://cheatsheetseries.owasp.org/
