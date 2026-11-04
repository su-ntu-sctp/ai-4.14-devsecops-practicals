# Assignment (Optional)

## Brief

Add non-blocking security checks to your repository with GitHub Actions, and practise managing a finding.

1. **Add OSV-Scanner with GitHub Actions**
   - Use your devops-demo repository (public) with the Lesson 4.13 CircleCI pipeline (a failing snyk_scan is expected)
   - Create `.github/workflows/osv-scanner.yml` that:
     - Runs on push to `main`, every week on a schedule, and manually (`workflow_dispatch`)
     - Uses Google's reusable workflow `google/osv-scanner-action/.github/workflows/osv-scanner-reusable.yml`
     - Pins it by **full commit SHA** (not by tag), with the version as a comment
     - Grants only `actions: read`, `contents: read` and `security-events: write`
   - Commit, push and check the results in the **Security → Code scanning** tab
   - Take screenshots showing:
     - The OSV-Scanner run in the Actions tab
     - The Security tab (alerts, or "no alerts")
   - Write a brief explanation (3-4 sentences):
     - Why this check is non-blocking, while `snyk_scan` in CircleCI is a gate
     - Why the workflow is pinned by commit SHA

2. **Triage a Finding**
   - Pick one LOW or MEDIUM OSV-Scanner alert from the Security tab
   - Create `osv-scanner.toml` with an `[[IgnoredVulns]]` entry that has an `id`, an `ignoreUntil` date and a `reason` that links your Lesson 4.13 security ticket
   - Push and confirm the chosen alert closes while the other findings stay open
   - Write a brief report (4-5 sentences):
     - The vulnerability ID, severity and fixed version (if any)
     - Why you accepted it temporarily, and who fixes it
     - When ignoring a finding is acceptable, and when it isn't
     - Why every ignore needs a reason and an expiry date

3. **Run a DAST Scan (Stretch)**
   - Add `.github/workflows/zap-baseline.yml` using `zaproxy/action-baseline` (pinned by commit SHA) against your live app URL (Render or Railway), stored as the repository variable `APP_URL`
   - Run it manually and download the `zap_scan` artifact
   - List the three highest-risk alerts and the solution ZAP suggests for each

## Submission (Optional)

- Submit the URL of the GitHub Repository that contains your work to NTU black board.
- Should you reference the work of your classmate(s) or online resources, give them credit by adding either the name of your classmate or URL.

## References
- OSV-Scanner: https://google.github.io/osv-scanner/
- OSV-Scanner GitHub Action: https://google.github.io/osv-scanner/github-action/
- OWASP ZAP baseline scan: https://www.zaproxy.org/docs/docker/baseline-scan/
- OWASP: https://cheatsheetseries.owasp.org/
